# Transactional Email API Alternatives: 4 Checks for Welcome Templates and Domain Verification

Short answer: for a generated property report, treat the email API response as acceptance of a delivery attempt, not proof that a manager received the attachment. Keep the report generation job separate from the send job, record the message identifier and delivery events, and give staff a way to see when a report needs another attempt. A welcome-email template and a verified sending domain matter, but neither closes that evidence gap.

Acceptance isn't delivery.

A property manager may request a monthly building report while a generation worker is still assembling the PDF. The safe path is to finish the file, validate its recipient and size, persist the report reference and dispatch intent, and only then ask a transactional email provider to send it. The provider's response moves the intent forward; later events determine whether the message was delivered, bounced, or left unresolved. This sequence also keeps a transient email outage from forcing an expensive report regeneration.

## How should a transactional email API alternative handle welcome templates and domain verification?

There are four useful states for this workflow: report ready, send accepted, delivery confirmed, and attention required. They describe application evidence, not an SMTP guarantee that a human opened or read a PDF. An API acknowledgement can mean the provider accepted a request for processing. A delivery event is stronger evidence of transport progress, yet it still does not prove that the attachment was read. Call it delivered in the operations view, not reviewed by the manager. The limitation is consequential: a delivery event says nothing about whether the property manager inspected the numbers, so any workflow that needs explicit approval must collect that approval separately.

Domain verification belongs before the first live send. Check the provider's required DNS records and confirm the sending domain passes its verification flow; also align the visible From domain with the authentication policy you intend to operate. Google's sender guidelines cover SPF, DKIM, DMARC, and authentication alignment. They are a better baseline for a domain checklist than a successful request in a local test. DNS propagation and policy changes make verification a deployment gate, not a checkbox in a template editor.

## Put the report behind a dispatch record

The example below leaves provider-specific transport behind an interface. It assumes that `report` already points to a durable PDF, and that a database transaction can atomically reserve a dispatch key before a network call. The dispatch key is based on the report revision and recipient: a correction to the PDF should be a new dispatch, while a repeated job for the same revision should not quietly send a second copy. The adapter must implement the provider's attachment encoding and size limits; those limits vary and need testing with representative files.

```ts
type Report = {
  id: string;
  revision: number;
  pdf: Uint8Array;
  recipient: string;
};

type Dispatch = {
  key: string;
  reportId: string;
  recipient: string;
  status: "ready" | "accepted" | "delivered" | "attention_required";
  messageId?: string;
};

interface DispatchStore {
  reserve(key: string, reportId: string, recipient: string): Promise<Dispatch>;
  markAccepted(key: string, messageId: string): Promise<void>;
  markAttentionRequired(key: string, reason: string): Promise<void>;
}

interface MailTransport {
  send(input: {
    to: string;
    subject: string;
    text: string;
    attachment: { filename: string; content: Uint8Array; contentType: string };
    dispatchKey: string;
  }): Promise<{ messageId: string }>;
}

async function sendReport(
  report: Report,
  store: DispatchStore,
  mail: MailTransport,
): Promise<void> {
  const key = `${report.id}:${report.revision}:${report.recipient.toLowerCase()}`;
  const dispatch = await store.reserve(key, report.id, report.recipient);
  if (dispatch.status !== "ready") return;

  try {
    const result = await mail.send({
      to: report.recipient,
      subject: `Report ${report.id} is ready`,
      text: "Your requested property report is attached.",
      attachment: {
        filename: `report-${report.id}.pdf`,
        content: report.pdf,
        contentType: "application/pdf",
      },
      dispatchKey: key,
    });
    await store.markAccepted(key, result.messageId);
  } catch (error) {
    await store.markAttentionRequired(key, String(error));
  }
}
```

That snippet sketches the boundary, not a complete exactly-once protocol. A worker can crash after the provider accepts the send but before `markAccepted` commits. If the transport supports an idempotency key, the adapter should pass `dispatchKey` using its documented mechanism. Otherwise an automatic replay of an ambiguous attempt can duplicate the attachment. Stop and reconcile the attempt using provider evidence or a manual review queue before retrying. That distinction is worth the extra state in a property workflow: a duplicate monthly report can confuse a recipient about which revision to retain. The trade-off is operational work: holding uncertain sends for review takes staff time, while blindly retrying risks an extra copy of a sensitive report. For a time-critical report, define who can authorize a replay and how they confirm the revision and recipient before pressing send again. A checksum of the PDF helps identify the exact artifact without putting its contents in the event log.

Don't retry blind.

## Test the failure path before choosing an API

Selection starts with a test matrix, not a template gallery. Send a representative PDF to controlled inboxes, including an address that rejects mail, and capture the API acknowledgement, message identifier, delivery or bounce event, and time of each transition. Test a delayed callback, a repeated callback, a worker restart after acceptance, an oversized attachment, and a report with a corrected revision. Never assume callback arrival order is stable. Authenticate webhook requests according to the chosen provider's documentation, store raw event IDs for deduplication, and make state transitions monotonic so a late acceptance record cannot overwrite a bounce.

Template editing is a separate concern. A welcome email may fit a hosted template with a name and signup link. A report attachment couples content to a generated artifact, a recipient, and a version; the template should control the short message around it, not conceal which file was sent. Keep recipient address, report revision, file checksum, provider message ID, and dispatch timestamps together in the application log, while limiting who can read the attachment or the logs. Do not log the PDF bytes. For SMS, a notification can tell staff to inspect a failed dispatch, but it is not a substitute for attaching the report to the requested email.

Provider comparisons should ask whether the service supports authenticated sending domains, attachment handling for the required file sizes, stable message identifiers, delivery and bounce events, event authenticity checks, and a documented path to investigate uncertain outcomes. SendGrid, Resend, and Postmark are three candidates to evaluate against the same test cases, not a ranking. Run those checks against each candidate's current documentation and a test account. Their product-specific attachment limits, domain-verification steps, and event interfaces need independent confirmation before a choice; a Node.js SDK call alone won't settle the delivery question. A smaller abstraction has its own limitation: it can hide provider features that matter during incident investigation. Expose the original provider message ID and raw event metadata alongside the portable dispatch state, with access controls for recipient details.

## Operate the handoff

Before release, verify the production sending domain, lock down access to stored PDFs, exercise the callback signature check, and rehearse the ambiguous-send case. Set an explicit age threshold for accepted messages without a terminal event and route those records to an operator; the threshold should follow the business deadline for the report, not an arbitrary retry timer. Keep the original dispatch record during recovery so staff can distinguish a replacement report from a duplicate attempt. Periodically inspect bounce reasons and authentication results. Recheck the whole path when the attachment size, template, sender domain, or provider changes.

This approach keeps cost and latency visible without letting either govern the decision. Generating the PDF once avoids repeated compute work; separating dispatch from generation avoids blocking the report request on email transport. Delivery evidence and a controlled replay policy are what make the attachment workflow operable.

## Sources

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- IETF, SMTP delivery status notifications (RFC 3461): https://www.rfc-editor.org/rfc/rfc3461
- IETF, DMARC (RFC 7489): https://www.rfc-editor.org/rfc/rfc7489
- Twilio, SMS documentation: https://www.twilio.com/docs/sms

## References

- https://support.google.com/a/answer/81126
- https://www.rfc-editor.org/rfc/rfc3461
- https://www.rfc-editor.org/rfc/rfc7489
- https://www.twilio.com/docs/sms
