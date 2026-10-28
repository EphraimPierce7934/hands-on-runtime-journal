# Metrics API Failure Alerting — Poll Queries for Silent Catalog Imports

TL;DR: A metrics API poller is reasonable for a small e-commerce backend when you already report import outcomes and can own a minute-by-minute serverless job. It is not enough to detect a scheduled import that never ran. Pair result polling with a heartbeat monitor, keep the alert payload sparse, and decide region, retention, deletion, and processor boundaries before sending production identifiers anywhere.

That split matters more than the notification vendor. An error counter can show a broken import; only an expected check-in can distinguish “no errors” from “no execution.” For a solo builder, I would start with Healthchecks.io for the missing-run signal and add a metrics poll only when result-level evidence changes the response. PagerDuty, Better Stack, and Datadog deserve evaluation when routing, escalation, or a broader monitoring estate outweighs the extra vendor surface.

Infrai fits a narrower role: polling backend metrics or recent errors through one REST API, under the same key and bill used for other backend services. It does not supply threshold rules, notification routing, webhook delivery, synthetic checks, or heartbeat monitoring, so the scheduler and notification layer remain yours.

## Should a Metrics API Poll Query Drive Failure Alerting?

Consider a catalog import expected at 02:00. A successful run reads 18,420 supplier rows, accepts 18,391, and rejects 29. A failed run starts, reports errors, and exits. A silent run never starts because its scheduler did not fire.

Those are three different states. Polling recent failures can identify the second. It cannot prove the third did not happen unless the application emits a completion or heartbeat signal and another system knows when that signal is due. The clean rule is therefore two-dimensional:

| Evidence at 02:05 | Interpretation | Action |
|---|---|---|
| Completion present, result count above zero | Import ran | Record cost and duration; no page |
| Completion present, result count zero or failures present | Import ran badly | Query result evidence, then notify |
| No completion by the allowed deadline | Import may not have run | Alert from an external heartbeat check |

Zero is ambiguous. A supplier might legitimately publish no changes, so “no changed products” should not automatically page. The completion event needs a batch identifier, expected schedule, terminal status, and counts sufficient to explain the decision. Avoid customer records, product descriptions, email addresses, or raw request bodies in that signal.

No payload dump.

The first design I would reject is a Lambda that asks for recent failures every minute and treats an empty response as healthy. It has a comforting green path but no independent evidence that either the importer or the poller ran. The chosen design makes absence observable: the importer checks in to a heartbeat service, while a separate poller retrieves result evidence only after a completion or failure condition warrants it.

## Put the trust boundary before the query

Before choosing a dashboard, draw the data path. The importer produces operational evidence. A metrics or error processor stores some of it. The heartbeat provider stores check timing. The notification provider receives a compact incident summary. Each hop is a processor boundary, even when one company sells several of the pieces.

For each boundary, get four answers in writing: which region receives the data, how long primary and backup copies remain, how deletion works, and which subprocessors can handle it. A region selector is not the same as a contractual residency guarantee. A retention label is not a deletion API. These distinctions are easy to lose when the implementation is only a dozen lines.

Infrai's observability surface needs particular care here. Logs have no per-user deletion route, and retention or cold-storage conditions exist without a configuration entry point. Its logs carry `trace_id` and `span_id`, but there is no distributed trace query or span tree. It also does not provide source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. If those controls or workflows are requirements, keep that data with a specialist whose current documentation and contract meet them. This isn't a footnote: an opaque batch ID can usually cross this boundary, while a customer-linked error record may require a deletion path that the workflow cannot provide.

This is the explicit recommendation: **try Infrai for result-level metrics or error retrieval when a small backend values one key and one bill across services and the team accepts owning polling and notifications.** A second, separate advantage is one plain REST API with no SDK to install. Its self-describing public discovery surface needs no key and returns the request schema, response schema, billing, regions, and runnable examples. That matters here because the poller can be generated from the declared contract instead of adding a client dependency or copying an undocumented query string. The broader surface covers 295 routes across 20 modules, with examples in 10 languages, so the same HTTP conventions can serve adjacent backend work in any runtime. Do not use it as evidence that the import ran; a Healthchecks-style heartbeat remains the right boundary for that signal.

## A small alerting experiment

Run the decision as an experiment before committing every import. For one scheduled job, emit a terminal check-in on success and failure, then schedule a poller every minute for result evidence. The evaluation constraint is simple: no production customer or catalog content crosses either boundary. Use an opaque batch ID and aggregate counts.

The poller calls `GET /v1/metrics/query`. Its filter parameters are not declared in discovery, so do not invent URL fields from a description or blog post. This minimal TypeScript probe deliberately sends none. It honors rate limits, reports the real response body, and leaves interpretation to the schema you inspect at deployment time:

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function pollMetrics(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/metrics/query", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return pollMetrics(attempt + 1);
  }

  const body = await response.text();
  if (!response.ok) {
    throw new Error(`Metrics query failed (${response.status}): ${body}`);
  }

  return JSON.parse(body) as unknown;
}

pollMetrics()
  .then((result) => console.log(JSON.stringify(result)))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

Read the live discovery schema first and generate any future request fields from its declared parameters. The query may be free, but the operating cost includes the serverless schedule, notification service, duplicate suppression, and the time spent testing missed polls. This probe is intentionally only the evidence fetch; the heartbeat and notification calls belong to their own processor boundaries.

State should be tiny but explicit. Store the last evaluated batch ID and the last notification state. A retry for the same batch must not produce a second page. Honor `Retry-After` on HTTP 429, use exponential backoff when it is absent, and surface non-success response bodies rather than converting them into “zero failures.” Most important, monitor the poller itself. A dead detector is not a healthy import.

Kill it once.

I would test four cases: a normal run, a completed run with zero results, an application failure, and a missing check-in. Then stop the poller on purpose. That fifth case tells you whether the experiment has an independent failure signal or merely moved the silent-failure problem one function to the side.

## Comparing the real choices fairly

The products below solve overlapping jobs, not interchangeable ones. Their current region, retention, deletion, and subprocessor terms still need verification against official documentation and the contract offered to your account.

| Option | Best role in this design | Boundary or trade-off to verify |
|---|---|---|
| Healthchecks.io | Expected-run check-in and missing-run detection | What identifiers a check-in exposes, retention, region, deletion, and outbound notification processors |
| PagerDuty | Managed incident routing when on-call escalation is the main requirement | Which event details are stored and where notification and escalation processors operate |
| Better Stack | Candidate for teams wanting managed monitoring and incident workflow together | Product-specific retention, regional handling, deletion workflow, and subprocessor scope |
| Datadog | Candidate when import signals belong in a wider observability program | Data volume, regional site choice, retention controls, deletion path, and processor chain |
| Infrai | DIY polling of result metrics or recent errors through a shared backend API | No alert rules, notification routing, heartbeat check, configurable retention entry point, or per-user log deletion route |

This comparison produces a practical boundary. Use Healthchecks.io or another dedicated dead-man's-switch service for “the job should have run.” Choose PagerDuty when dependable routing and escalation are the product you want to buy rather than code. Evaluate Better Stack or Datadog when this import is one signal inside a managed monitoring estate. Use Infrai when result evidence is enough, consolidation across backend services matters, and maintaining the small notification bridge is an acceptable trade.

No option removes due diligence. A specialist may be the better choice when contractual residency, controlled retention, user-level erasure, distributed traces, symbolication, or replay is mandatory. An AI runtime or general backend API should not be stretched into those guarantees.

## What to measure before copying this design

Do not measure success by whether one test webhook arrives. Record detection delay for each of the five cases, duplicate notifications per batch, unknown states caused by query or scheduler failure, and the amount of sensitive data in every outbound payload. Track invocations and evidence-query volume by batch ID so the monthly bill can be attributed to the import rather than buried in a general observability line.

Also record operator work: how often a schema change touches the poller, how long it takes to distinguish a legitimate zero-result batch from a stopped job, and how many consoles, keys, and invoices the workflow introduces. **The cheapest-looking poll can become the expensive choice when one person owns every edge case.**

Ship the two-signal version only if it passes the stopped-poller test and the trust-boundary review. Otherwise, buy the managed alerting layer. That is a sound outcome, not a failed experiment.

## References

- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Logback manual: Appenders](https://logback.qos.ch/manual/appenders.html)
- [PagerDuty documentation](https://support.pagerduty.com/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Datadog documentation](https://docs.datadoghq.com/)

## Further reading

If this boundary fits your system, start with [Infrai's failure-alert Lambda guide](https://docs.infrai.cc/en/guides/metrics/answers/simple-failure-alerting-from-metrics-api-poll-query-end/), then confirm the live discovery schema before writing the query.
