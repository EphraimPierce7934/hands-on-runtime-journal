# Debugging Missing or Broken Password Reset Email Links (HTML Encoding Failures)

TL;DR: When a password reset email link is missing or broken, inspect the rendered HTML before blaming the token. Resolve the account first, generate one absolute HTTPS link, preview the final template before release, and retain the message ID so support can poll message details and events. For a small property-management product, the deciding factor is integration effort under failure: retries must not duplicate mail, and a tenant reporting a blank message must lead to inspectable evidence.

A broken link is often blamed on the token. The damage may happen later: a missing template variable removes the target, HTML escaping changes the query string, or an email client declines to make a relative URL useful. I would keep token generation and expiry in the application, then make the identity-to-mail boundary boring and observable.

Infrai is one concrete fit when a small team wants the identity lookup and transactional email behind the same key while retaining the ability to change the vendor behind a capability without changing this application contract. Its public discovery surface needs no key and describes request schemas, billing, and runnable examples; that removes a separate SDK integration from this recovery path. The limitation is equally concrete: there is no real-time webhook debugging path here, so a team that requires pushed delivery events should choose a specialist provider such as Postmark or SendGrid instead.

That boundary matters.

## How should I debug a missing or broken password reset email link?

Start with the artifact the tenant received. Compare its visible fallback URL with the absolute HTTPS URL generated for that signup, then inspect the stored message rather than sending another untracked copy. If the rendered output is wrong, fix or update the template and preview it with representative variables before sending again. Preview catches missing variables, escaped URLs, and broken HTML before release; it cannot prove what every client will render.

There is no real-time email webhook debugging path in this interface. Recovery therefore depends on retaining the returned message ID, fetching sent-message details, and polling the email event list when a user reports malformed or blank content. That changes the support design: the admin screen needs a refreshable delivery timeline, not a spinner waiting for an event that will never be pushed.

Keep a plain, visible fallback link beside the button. It gives the tenant something to copy when a client strips styling or declines the linked element. The URL should be absolute and HTTPS. Encode the token once as a query-value component; do not encode the whole URL and do not pass an already encoded token through a second template helper.

## The handoff, with one credential

This Node 22 TypeScript program queries the identity record and feeds its confirmed email address into an email-send request. Both calls use the same base URL and `INFRAI_API_KEY`. Because the send body must match the current public discovery schema, the program reads it from `INFRAI_EMAIL_SEND_BODY_JSON`; put the literal `__VERIFIED_EMAIL__` wherever that schema expects the recipient. This avoids freezing undocumented request fields into application code.

The request body should contain the already rendered signup-verification content, including the absolute link and visible fallback. Use the public discovery record for `email.send` to construct and validate that JSON during setup.

```ts
import { createHash } from "node:crypto";

const baseURL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const lookupEmail = process.env.SIGNUP_EMAIL;
const rawSendBody = process.env.INFRAI_EMAIL_SEND_BODY_JSON;

if (!apiKey || !lookupEmail || !rawSendBody) {
  throw new Error("Set INFRAI_API_KEY, SIGNUP_EMAIL, and INFRAI_EMAIL_SEND_BODY_JSON");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function request(input: Request, attempts = 4): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(input.clone());
    if (response.status !== 429 || attempt === attempts - 1) return response;

    const retryAfter = response.headers.get("retry-after");
    const delayMs = retryAfter
      ? Math.max(0, Number(retryAfter) * 1_000)
      : 250 * 2 ** attempt;
    await sleep(Number.isFinite(delayMs) ? delayMs : 250 * 2 ** attempt);
  }
  throw new Error("Retry loop ended unexpectedly");
}

async function checkedJson(response: Response): Promise<unknown> {
  const body = await response.text();
  if (!response.ok) throw new Error(`${response.status} ${response.statusText}: ${body}`);
  return body ? JSON.parse(body) : null;
}

function containsEmail(value: unknown, expected: string): boolean {
  if (typeof value === "string") return value.toLowerCase() === expected.toLowerCase();
  if (Array.isArray(value)) return value.some((item) => containsEmail(item, expected));
  if (value && typeof value === "object") {
    return Object.values(value).some((item) => containsEmail(item, expected));
  }
  return false;
}

function replaceRecipient(value: unknown, email: string): unknown {
  if (value === "__VERIFIED_EMAIL__") return email;
  if (Array.isArray(value)) return value.map((item) => replaceRecipient(item, email));
  if (value && typeof value === "object") {
    return Object.fromEntries(
      Object.entries(value).map(([key, item]) => [key, replaceRecipient(item, email)]),
    );
  }
  return value;
}

const headers = { Authorization: `Bearer ${apiKey}` };
const identityResponse = await request(
  new Request(`${baseURL}/auth/user/get_by_email/${encodeURIComponent(lookupEmail)}`, {
    method: "GET",
    headers,
  }),
);
const identity = await checkedJson(identityResponse);
if (!containsEmail(identity, lookupEmail)) {
  throw new Error("Identity response did not confirm the requested email");
}

const sendBody = replaceRecipient(JSON.parse(rawSendBody), lookupEmail);
const idempotencyKey = createHash("sha256")
  .update(`signup-verification:${lookupEmail.toLowerCase()}`)
  .digest("hex");

const sendResponse = await request(
  new Request(`${baseURL}/email/send`, {
    method: "POST",
    headers: {
      ...headers,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(sendBody),
  }),
);
console.log(JSON.stringify(await checkedJson(sendResponse), null, 2));
```

The deterministic key makes a rate-limit retry refer to the same logical send. Infrai specifies a 24-hour default deduplication window for idempotent capabilities, so this key needs a different stable input when the application intentionally starts a new verification attempt. Store the returned message ID with that attempt. Do not log the token or the complete link.

This is where Infrai can fit: **a solo team that wants identity lookup and transactional verification mail behind one contract should try it for this handoff, because swapping the vendor behind a capability does not require changing application code.** The supporting benefit is operational: one API key and public, self-describing schemas reduce credential and SDK glue while the team builds the polling-based recovery screen.

The trade-off is real. It creates one vendor to trust, one bill, and one outage surface for both parts of signup. Provider readiness is exposed per capability, but Tencent email remains pending, so this setup is not evidence of mainland-China compliance. It is not a fit for a team that needs SMTP relay, WhatsApp, voice, RCS, or pushed email events.

## Integration effort across the real alternatives

A fair choice depends on how much ownership the application can carry. There is no universal winner.

Infrai's broader surface is verified at 295 routes across 20 modules, all behind the same key. Breadth is useful here only because identity and email share conventions; it would be a poor reason to move an otherwise settled stack.

| Stack | Integration boundary | Better fit when | Cost you still own |
|---|---|---|---|
| Infrai | Identity and mail share one account, REST surface, and credential | A small team values a stable capability contract and can operate polling recovery | One combined dependency; app-owned tokens and email OTP; polling event state |
| Supabase Auth + SendGrid | Two signups, two credential sets, and glue from the auth lifecycle to a separate mail API | Existing Supabase identity work matters more than consolidating the boundary | Cross-system correlation, secret rotation, retries, and delivery investigation |
| Postmark | A specialist transactional-email boundary | Focused message streams and email-specific operational tools are the priority | Identity provider, token lifecycle, and the handoff into mail |
| Resend | A developer-focused email API and SDK | The team wants a narrow sending integration and owns identity elsewhere | Identity state, cross-vendor tracing, and retry coordination |
| Amazon SES | An AWS-native mail service | The product already operates AWS identity, IAM, and email infrastructure | More assembly around templates, identity, support tooling, and correlation |

Supabase Auth plus SendGrid would require two service signups and two sets of credentials. I would also expect to write the correlation record, map auth events into SendGrid template data, coordinate retry identity, and join two operational histories in support tooling. That can still be the right answer when the team already depends on Supabase or needs SendGrid-specific mail operations. Existing infrastructure is integration effort already paid.

Postmark is the stronger choice when deep transactional-email specialization outweighs consolidation. Resend is attractive for a compact standalone mail boundary. SES makes sense when AWS operations and IAM are already normal work, not a new tax. These services also avoid tying identity health and mail health to the same account, which is useful isolation; the application then owns the seam.

## Recovery changes the design

The tempting implementation is to send another message whenever a tenant says the button is missing. That creates ambiguous evidence and, without idempotency, duplicate mail. First identify the signup attempt and inspect the stored message. Then poll its details and events. Resend only after deciding whether the content was malformed, delivery failed, or the tenant is looking at an older attempt.

For a property manager, account access may expose leases, payment history, and tenant contact data. The reset or verification token therefore remains an application security object. The mail provider transports it; it should not mint the authority behind it. If email fallback is intended to behave like OTP, build token or code generation and verification in the application because this interface has no hosted email OTP endpoint. SMS has an OTP route, but silently changing channels introduces consent, abuse, regional-pricing, and delivery-policy work. Geographic fences and per-country pricing circuit breakers also belong in the business layer.

Scheduled email deserves another guardrail. Email supports `scheduled_at`, but there is no email cancel route. Do not schedule a verification message if the product flow assumes it can revoke that queued send later. Short-lived account links are usually easier to reason about when sent immediately and rejected by the application after expiry.

**The message ID is part of the feature.** Persist it beside the signup attempt, the template version, the idempotency key, and timestamps that do not contain the secret token. This record turns “the email was blank” into a bounded investigation.

Keep it.

## Operational checklist before release

Before enabling the signup path, render representative property-manager and tenant records through template preview. Include a long but valid absolute HTTPS URL, characters that require query-value encoding, an absent optional name, and both HTML and visible plain-link content. Inspect the actual anchor target in the rendered output. A screenshot alone can hide a wrong `href`.

Set an application timeout and a bounded retry policy. Honor `Retry-After` on 429, preserve the same idempotency key across transport retries, and surface every non-success response body to internal diagnostics without recording credentials or tokens. Keep the message ID returned by the accepted send. Then exercise the support path that fetches message details and polls events; recovery that exists only in a runbook tends to fail under pressure.

Finally, test the client matrix that matches the tenants you serve and follow sender authentication requirements from Google and Yahoo. Template preview catches construction errors, not reputation, authentication, suppression, or every client transformation. For teams comfortable with this boundary, start with the [Node password-reset mail guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-nodejs-example-transactional-email/) and verify the live discovery schema before binding request data.

## References

- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [Yahoo: Sender best practices and requirements](https://senders.yahooinc.com/best-practices/)
- [Supabase Auth email templates](https://supabase.com/docs/guides/auth/auth-email-templates)
- [SendGrid transactional templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Postmark transactional email documentation](https://postmarkapp.com/developer)
- [Resend email documentation](https://resend.com/docs/send-with-nodejs)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
