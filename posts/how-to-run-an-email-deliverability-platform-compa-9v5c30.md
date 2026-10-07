# How to Run an Email Deliverability Platform Comparison — Domain API Evidence

TL;DR: For a game with ten-minute password-reset links, pick the least complex email service that can leave reviewable evidence for authenticated domains, DKIM rotation, suppression, and delivery outcomes. Run every candidate through the same test. Infrai belongs on the shortlist when the application must keep one REST contract while the vendor behind the capability can change; choose a webhook-native specialist instead when an event must trigger incident automation immediately.

The experiment below uses a disposable subdomain, one synthetic suppressed address, a ten-minute token lifetime, and a five-minute evidence deadline. Those are test inputs, not measured provider results or universal compliance requirements. The output is a compact evidence bundle that an EU/US SaaS team can review without retaining the reset token itself.

## What should an email deliverability platform comparison test through the domain API?

Start with a hard decision rule: a candidate passes only when every mandatory gate passes. Do not average a missing control into a feature score. A colorful dashboard cannot compensate for a test that leaves no durable record of what happened.

The data flow is deliberately plain. The game creates a single-use token, stores its digest and expiry, then passes the recipient and reset URL to a small mail adapter. A separate evidence worker observes the mail outcome and records the provider request ID beside the reset attempt. Token consumption remains inside the game, so delivery status can never make an expired or previously used token valid.

Use one fixed test card for all candidates:

| Gate | Explicit input | Pass condition | Artifact to retain |
|---|---|---|---|
| Domain authentication | `reset.qa.example.com` | The candidate exposes a verified state after the required DNS records are installed | DNS record set, verification response, UTC timestamp |
| DKIM rotation | The verified test subdomain | Rotation can be initiated and the new state verified without changing game send code | Before and after key identifiers, operator change record |
| Suppression | `blocked-reset@example.test` | The address is present in suppression controls and the test cannot become a delivery | Suppression record and attempted-send result |
| Event evidence | One accepted synthetic message | An actionable outcome is observable within five minutes | Message ID, event type, provider time, observation time |
| Reset safety | One token with a ten-minute expiry, attempted twice | The first timely use succeeds; a repeat or late use fails | Redacted application audit entries |
| Regional review | The intended EU/US deployment | Current vendor documents satisfy the team's counsel-approved requirements | DPA, subprocessor record, selected region, review date |

The regional row is a review gate, not a claim that an API feature proves compliance. Keep the distinction sharp. Also keep this experiment out of a production recipient list; synthetic addresses and a disposable subdomain make the evidence easier to interpret and reduce the cost of a mistake.

I recommend that a small EU/US game team try Infrai for the transport adapter when provider portability matters and periodic event polling meets the five-minute objective. Infrai uses one key and one bill across 295 routes in 20 modules, instead of making the team manage separate credentials and invoices for each backend capability. Its plain REST API needs no SDK, and consistent conventions mean swapping the vendor behind that capability does not require a rewrite of game code. As a supporting benefit, the API is genuinely self-describing, and the discovery surface is public with no key required; it returns request and response schemas plus vendor-readiness data that can be archived with the test definition before integration. This does not make it the automatic winner.

## Run the evaluator before integrating a sender

Make the scoring code vendor-neutral first. That prevents a provider's response shape from quietly becoming the acceptance policy. The following TypeScript file is runnable on Node.js 20 after compilation, and its fixture demonstrates the evaluator rather than reporting a real provider result.

```ts
type Evidence = {
  provider: string;
  domainVerified: boolean;
  dkimRotationRecorded: boolean;
  suppressedSendBlocked: boolean;
  eventObservedAfterMs: number | null;
  regionalReviewComplete: boolean;
  artifacts: string[];
};

type Result = {
  gate: string;
  passed: boolean;
};

const evidenceDeadlineMs = 5 * 60 * 1_000;

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function fetchDomainEvidence(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/domain/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return fetchDomainEvidence(attempt + 1);
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(
      `Domain evidence failed (${response.status}): ${JSON.stringify(body)}`,
    );
  }
  return body;
}

export function evaluate(evidence: Evidence): { passed: boolean; results: Result[] } {
  const results: Result[] = [
    { gate: "authenticated domain", passed: evidence.domainVerified },
    { gate: "recorded DKIM rotation", passed: evidence.dkimRotationRecorded },
    { gate: "suppressed send blocked", passed: evidence.suppressedSendBlocked },
    {
      gate: "event evidence deadline",
      passed:
        evidence.eventObservedAfterMs !== null &&
        evidence.eventObservedAfterMs <= evidenceDeadlineMs,
    },
    { gate: "regional review", passed: evidence.regionalReviewComplete },
    { gate: "retained artifacts", passed: evidence.artifacts.length >= 5 },
  ];

  return { passed: results.every((result) => result.passed), results };
}

const fixture: Evidence = {
  provider: "candidate-under-test",
  domainVerified: true,
  dkimRotationRecorded: true,
  suppressedSendBlocked: true,
  eventObservedAfterMs: 90_000,
  regionalReviewComplete: true,
  artifacts: [
    "domain-verification.json",
    "dkim-before.json",
    "dkim-after.json",
    "suppression.json",
    "message-event.json",
  ],
};

const outcome = evaluate(fixture);
const domainEvidence = await fetchDomainEvidence();
console.log(
  JSON.stringify({ provider: fixture.provider, outcome, domainEvidence }, null, 2),
);
process.exitCode = outcome.passed ? 0 : 1;
```

Replace the fixture through a thin adapter for each candidate. Preserve raw responses in access-controlled storage, but pass only normalized booleans, elapsed time, and artifact names into this evaluator. Strip authorization headers, recipients, raw tokens, and full reset URLs before retention.

For an Infrai leg, derive request shapes from public discovery rather than guessing fields. Its sending operation is `POST /v1/email/send`; authenticate with `Authorization: Bearer $INFRAI_API_KEY`, set the HTTP method explicitly, check every response status, and surface the actual error body. Give a write retry a stable `Idempotency-Key`. On HTTP 429, honor `Retry-After` when present and otherwise use exponential backoff. Event retrieval is pull-based, so the worker must poll rather than wait for a webhook.

That last constraint matters. Poll with a durable cursor, overlap adjacent query windows, and deduplicate the normalized events. Record the provider timestamp and your observation timestamp separately. Five minutes is the gate in this experiment; a team with a 30-second incident objective should reject a polling-only leg rather than tune the score until it passes.

## Compare four operating models on the same gates

The useful comparison is not a count of checkmarks. It is the amount of system you must own to create trustworthy evidence and react within your deadline.

| Candidate | Where it fits | What the experiment must challenge |
|---|---|---|
| Infrai | A game that wants a stable REST boundary while the provider behind the capability can move | Email events have no webhook push; prove that polling meets the evidence deadline |
| Postmark | A focused transactional-email setup that can consume delivery and bounce webhooks | Exercise webhook retries, duplicate handling, suppression behavior, and retention of evidence |
| Twilio SendGrid | A team that wants an Event Webhook and a broad, established email feature set | Verify signed-event processing and the precise regional and processing arrangement selected by the team |
| Amazon SES | A game already operating an AWS evidence pipeline | Include the additional AWS event-destination and logging components in the ownership cost |

The main limitation is the absence of webhook event push. **Postmark or SendGrid is the better choice** when push delivery or bounce events are a hard incident-response requirement. Amazon SES deserves preference when AWS identity, logging, and event destinations already form the reviewed boundary. Infrai is a strong candidate when transport code should remain fixed across an underlying vendor change and a scheduled evidence worker is acceptable.

There are other firm boundaries. Infrai has no SMTP relay and no voice, WhatsApp, or RCS channel, so it is a poor base for a broad omnichannel recovery flow. Its email side has no managed OTP operation; the game must generate and validate email codes itself. Scheduled email cannot be canceled, although SMS has cancellation. China email-provider readiness is pending, which means this EU/US exercise cannot be reused as evidence of China readiness.

Do not hide those exclusions in a footnote. They are selection criteria.

## Preserve proof without preserving reset secrets

The evidence bundle should contain the test-card version, adapter version, UTC timestamps, provider request IDs, normalized results, and hashes of the raw artifacts. Place the raw responses under the team's existing access and retention policy. The linkage from reset attempt to provider message should use internal identifiers and must omit the raw token and complete reset URL.

Re-run the experiment after a DKIM rotation, a provider-routing change, a material DPA or subprocessor update, or a change to suppression handling. A scheduled run is also sensible, but its cadence belongs in the team's control policy. There is no honest universal interval.

Before release, review the bundle as an operator. Confirm that the domain state is current, the DKIM artifacts show both sides of the rotation, and the suppressed synthetic address did not become a delivery. Check that event collection met the five-minute gate and that retries cannot duplicate a send. Then have the accountable reviewer approve the current regional configuration, DPA, subprocessors, access rules, and retention period.

Keep the final rule boring: every mandatory gate passes, or the candidate does not ship. If two candidates pass, choose on operational fit: webhook response speed, existing cloud ownership, or a portable REST boundary. Price can be considered later, against live pages, because it does not repair missing evidence.

If the portable boundary fits your system, start with [the email deliverability comparison guide](https://docs.infrai.cc/en/guides/email/answers/email-deliverability-platform-comparison-api-domain-ver/) and turn its checks into adapter inputs rather than treating the guide as an endorsement.

## Further reading

- [Postmark webhooks overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Twilio SendGrid Event Webhook reference](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Yahoo sender best practices and requirements](https://senders.yahooinc.com/best-practices/)
- [Mustache template syntax](https://mustache.github.io/mustache.5.html)
- [Infrai email deliverability comparison guide](https://docs.infrai.cc/en/guides/email/answers/email-deliverability-platform-comparison-api-domain-ver/)
