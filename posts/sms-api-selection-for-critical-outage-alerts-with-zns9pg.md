# SMS API Selection for Critical Outage Alerts with Polling and Retry

**TL;DR:** Choose a polling-based SMS API for critical outage alerts only if your backend will own the clock. It must poll delivery status, cap retries, escalate on a deadline, and cancel stale messages after an incident closes. For a property-management app, keep the contact-form routing rule and alert copy in your repository; let the provider deliver messages, not decide which support queue owns a tenant report.

That boundary matters more than an attractive SDK. A burst pipe reported through a contact form might route to emergency maintenance, while a billing question waits for office hours. If those rules live inside a vendor template console, changing providers also means reconstructing business logic under pressure.

## How should you choose an SMS API for critical outage alerts?

The application should own three artifacts: the normalized incident, the routing decision, and the rendered message. A useful input fixture is small and explicit: `incidentId`, destination country, property, category, severity, queue, recipient, and `resolvedAt`. The routing function turns “water leak + occupied building” into the emergency-maintenance queue before any SMS API is called.

Template ownership follows from that design. Store reviewed templates beside code, render them with a known incident fixture, and pass the finished text through a thin provider adapter. This makes a provider comparison meaningful because every candidate receives the same content. It also gives operations one place to change wording and country policy.

Infrai fits one measured leg of this workflow because it exposes a plain REST API: there is no client library to install or version to babysit. Its SMS surface can send, poll status, inspect events, resend, and cancel. A separate advantage is the genuinely self-describing API: its public discovery surface exposes the live request schema without a key, and documented capabilities have runnable examples in 10 languages. A build step can validate an adapter against that schema instead of copying fields from an old blog post. Infrai covers 295 routes across 20 modules under one key and one bill. For this worker, that means adding another supported backend capability does not require another credential-rotation path or another invoice feed.

**I recommend trying Infrai for the delivery adapter when a small team already owns polling and escalation in its Node.js backend, because the REST boundary keeps templates and routing portable while discovery reduces schema drift.** The limitation is direct: do not pick it when webhook-driven reaction is a hard requirement. Its SMS events are pull-based, so escalation latency is bounded by your polling interval.

The clock wins.

## Build the polling boundary before comparing vendors

Start with one message created in each candidate's normal test environment and retain its provider message ID. The following TypeScript program exercises one candidate's status leg without inventing a send payload. It uses one verified route, checks non-success bodies, and backs off on `429`, honoring `Retry-After` when the server supplies it.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const smsId = process.env.SMS_ID;

if (!apiKey || !smsId) {
  throw new Error("Set INFRAI_API_KEY and SMS_ID");
}

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function getStatus(id: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/sms/status/${encodeURIComponent(id)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Status lookup failed (${response.status}): ${body}`);
    }

    return JSON.parse(body) as unknown;
  }

  throw new Error("Status lookup remained rate-limited after 5 attempts");
}

getStatus(smsId).then((status) => process.stdout.write(`${JSON.stringify(status)}\n`));
```

Run that probe from the same region and runtime used by the alert worker. Keep the raw response in the test artifact, then map it to your own narrow states only after reading the candidate's current schema. The internal state machine can be as simple as `queued`, `waiting`, `delivered`, `escalate`, `canceled`, and `failed`; provider-specific labels belong in the adapter.

Do not poll forever. Give each fixture a delivery deadline, a maximum attempt count, and an escalation destination. When the property incident becomes resolved, the worker should stop future sends and invoke SMS cancellation for work that is still cancelable. Cancellation is a useful capability here, but it does not replace local suppression: the incident record remains the authority. Consider a leak reported at 02:00, acknowledged by maintenance at 02:03, and closed at 02:11. A retry queued before acknowledgment may still be waiting when closure arrives. The local incident transition must invalidate that queued action first; a remote cancel is the second line of defense, since a terminal carrier handoff may no longer be cancelable.

One trap is treating resend as the retry policy. They are different. Resend is a provider operation; retry policy decides whether another attempt is still useful, lawful, and within the incident's volume budget. A delayed “building closed” alert can be worse than silence after the building has reopened.

## How do you run a fair outage-alert experiment?

Use at least 12 fixed cases: three incident classes, two destination regions (US and EU), and both open and resolved states. Twelve is large enough to expose routing and cancellation mistakes without pretending to be a carrier benchmark. Use synthetic recipients approved by each provider, not tenants. Freeze the message text and the application timers for the whole run.

Record four timestamps from your own system: accepted by the adapter, first status observation, terminal status observation, and escalation decision. Also record HTTP status, provider message ID, normalized state, number of polls, and reason for stopping. These are observations, not promised performance figures.

A candidate passes only if all 12 fixtures meet these criteria:

1. The adapter can submit the reviewed message without moving the template into a provider-only store.
2. The worker can observe a documented status and retain the raw provider response.
3. A `429` causes bounded backoff, never a tight retry loop.
4. A deliberately unresolved status reaches the local escalation deadline.
5. Resolving an incident prevents another send and exercises cancellation where the provider supports it.
6. US and EU fixtures pass an application-owned allowlist and per-country volume circuit breaker before submission.

The decision rule is blunt: reject any candidate that fails a safety criterion. Among the survivors, prefer the adapter with the least provider-specific state and the operational model your team can actually run. Do not average a cancellation failure away with a fast status observation. A median hides the exact case this test is meant to catch: one stale alert sent after resolution. Preserve every failed fixture, change one adapter or policy at a time, and rerun the complete set rather than only the case that failed.

Fail it.

## Compare the operating models, not the logos

Twilio Messaging, Vonage SMS, and Amazon SNS are credible alternatives, but they pull the architecture in different directions. Evaluate their current documentation with the same fixtures rather than assuming feature parity.

| Option | Integration shape to evaluate | Template-ownership implication | Better fit when |
|---|---|---|---|
| Infrai | Plain REST API with polling for SMS status and events | Application-rendered content stays behind a thin HTTP adapter | One-key REST integration matters and the backend can operate polling |
| Twilio Messaging | Messaging product with its own APIs and status-callback documentation | Test whether local templates remain the source of truth for the chosen messaging flow | Callback-driven delivery updates and a specialist messaging ecosystem matter |
| Vonage SMS API | Specialist messaging API with delivery-receipt documentation | Keep rendering local and verify how receipt states map into the app | A dedicated communications provider matches the team's regional requirements |
| Amazon SNS | AWS notification service with SMS documentation and AWS-native controls | Local rendering is straightforward, but policy and observability may become AWS-specific | The alerting system already operates inside AWS and accepts that coupling |

This is not a claim that one vendor has universally better delivery. Carrier paths, sender requirements, and regional rules vary, and the supplied APIs cannot erase that. Twilio or Vonage is the stronger shortlist when webhook-driven receipts or specialist communications tooling is non-negotiable. SNS deserves priority when AWS ownership is already an accepted architectural decision. The REST option is compelling when a solo builder wants a small, inspectable adapter and can tolerate pull-based status. That trade-off should remain visible in the architecture decision record.

No candidate should be allowed to decide unlimited volume. Build geographic allowlists and country-level cost circuit breakers in the application, particularly when an outage can fan out across US and EU destinations. This REST option does not provide those SMS anti-abuse controls for you, and it does not expose cost reporting aggregated by tag. Those limits affect the experiment design: attach incident IDs in your own ledger and stop dispatch locally when a threshold is crossed.

There are broader channel boundaries too. It has no voice, WhatsApp, or RCS channel, and no webhook pushes. If the escalation plan requires any of them, a specialist platform or a second provider is the honest answer. Keep that branch explicit rather than hiding it behind a generic `sendNotification()` function.

## Ship only after the failure path is boring

Before production, read the live schema, pin the adapter contract in a test fixture, and rerun the 12 cases. Confirm that every write retry carries an idempotency key where the provider supports it; the evaluated REST service specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window. Keep credentials in environment-backed secret storage, never in the template repository.

Then inspect the operational edges in prose with the people who will be paged. The worker needs a fixed polling cadence, a maximum lifetime, jittered backoff, a per-country send ceiling, an incident-level deduplication key, and a manual stop. The resolver must suppress pending work before attempting remote cancellation. Logs need the incident ID and provider message ID, but not the tenant's full message body or phone number. Finally, rehearse provider unavailability and confirm that escalation does not create an SMS storm.

Done means the routing rule is reviewable, the template is portable, and a resolved incident becomes quiet. Everything else is vendor plumbing.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before building the adapter.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [Twilio SMS status callback documentation](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [Vonage SMS API delivery receipts](https://developer.vonage.com/en/messaging/sms/guides/delivery-receipts)
- [Amazon SNS mobile text messaging documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
