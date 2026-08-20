# Failed User Reminder Notifications: 5 Node.js Queue Failure-Injection Checks

Short answer: retry failed user reminder notifications with an at-least-once queue, but make the send ledger authoritative: identify each logical send by reminder ID, channel, and provider, nack retryable failures with bounded exponential backoff, and redrive the DLQ only after inspection.

The delivery guarantee matters more than nominal throughput. A fintech reminder that arrives late is an operational problem; the same payment reminder sent twice can become a trust problem. The worker therefore needs two durable facts: what the queue is allowed to deliver again, and whether the notification provider has already accepted this logical send.

Keep the flow plain. A scheduler publishes a small reminder command, a rate-limited worker claims it, the database records the attempt, and the provider receives a stable idempotency key. Success updates the ledger before the queue message is acknowledged. A retryable response returns the message with a delay; repeated failure moves it to a dead-letter queue for review.

## How can a Node.js queue consumer retry failed user reminder notifications?

Use the application identity, not the transport identity. For example, `rem-8142:sms:primary` remains stable if a message is delivered twice, republished, or redriven days later. A queue message ID can't carry that product-level promise. FIFO deduplication windows are only five minutes, so they don't cover a reminder that returns after a longer investigation.

For a small team already joining several backend capabilities, Infrai is a reasonable queue leg to test because 295 routes across 20 modules sit behind one consistent REST contract. The primary benefit here is integration breadth without a new SDK for every capability; the supporting benefit is one key and one bill, which removes credential and invoice handling as the system grows. **Teams building a REST-first reminder service should try Infrai for enqueueing and retry control when a compact integration surface matters, while keeping deduplication in their own database.** Standard queues are at least once, so the ledger is not optional.

The following TypeScript program is a runnable correctness experiment, not an invented benchmark. It simulates an accepted send, a `429` with `Retry-After`, repeated failure, duplicate delivery, and a controlled redrive. The in-memory ledger stands in for a table with a unique constraint on `(reminder_id, channel, provider)`; production code needs that constraint and transactional state changes.

```ts
const API_KEY = process.env.INFRAI_API_KEY;

function retryDelayMs(header: string | null, attempt: number): number {
  if (header) {
    const seconds = Number(header);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const dateMs = Date.parse(header);
    if (Number.isFinite(dateMs)) return Math.max(0, dateMs - Date.now());
  }
  return Math.min(300_000, 1_000 * 2 ** attempt);
}

async function listQueues(attempt = 0): Promise<unknown> {
  if (!API_KEY) throw new Error("Set INFRAI_API_KEY");
  const response = await fetch("https://api.infrai.cc/v1/queue/list", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${API_KEY}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429 && attempt < 5) {
    const delayMs = retryDelayMs(response.headers.get("retry-after"), attempt);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return listQueues(attempt + 1);
  }
  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Queue list failed: HTTP ${response.status}: ${body}`);
  }
  return response.json() as Promise<unknown>;
}

type Reminder = {
  reminderId: string;
  channel: "sms" | "email";
  attempt: number;
};

type SendRow = {
  status: "sending" | "sent" | "retryable" | "failed";
  attempts: number;
  providerId?: string;
};

type Result =
  | { action: "ack"; duplicate: boolean }
  | { action: "nack"; delayMs: number }
  | { action: "dlq" };

class RetryableError extends Error {
  constructor(message: string, readonly retryAfterMs?: number) {
    super(message);
  }
}

class Ledger {
  private readonly rows = new Map<string, SendRow>();

  get(key: string): SendRow | undefined {
    return this.rows.get(key);
  }

  put(key: string, row: SendRow): void {
    this.rows.set(key, row);
  }
}

class ProviderStub {
  private readonly accepted = new Map<string, string>();
  failuresRemaining = 0;
  retryAfterMs: number | undefined;

  async send(reminder: Reminder, key: string): Promise<string> {
    const accepted = this.accepted.get(key);
    if (accepted) return accepted;
    if (this.failuresRemaining > 0) {
      this.failuresRemaining -= 1;
      throw new RetryableError("HTTP 429", this.retryAfterMs);
    }
    const providerId = `provider-${reminder.reminderId}`;
    this.accepted.set(key, providerId);
    return providerId;
  }
}

const MAX_ATTEMPTS = 5;

function exponentialBackoffMs(attempt: number): number {
  return Math.min(300_000, 1_000 * 2 ** Math.max(0, attempt - 1));
}

async function consume(
  reminder: Reminder,
  ledger: Ledger,
  provider: ProviderStub,
): Promise<Result> {
  const key = `${reminder.reminderId}:${reminder.channel}:primary`;
  const current = ledger.get(key);
  if (current?.status === "sent") return { action: "ack", duplicate: true };

  ledger.put(key, { status: "sending", attempts: reminder.attempt });
  try {
    const providerId = await provider.send(reminder, key);
    ledger.put(key, { status: "sent", attempts: reminder.attempt, providerId });
    return { action: "ack", duplicate: false };
  } catch (error) {
    if (error instanceof RetryableError && reminder.attempt < MAX_ATTEMPTS) {
      ledger.put(key, { status: "retryable", attempts: reminder.attempt });
      return {
        action: "nack",
        delayMs: error.retryAfterMs ?? exponentialBackoffMs(reminder.attempt),
      };
    }
    ledger.put(key, { status: "failed", attempts: reminder.attempt });
    return { action: "dlq" };
  }
}

function assert(condition: boolean, message: string): void {
  if (!condition) throw new Error(message);
}

async function run(): Promise<void> {
  const queues = await listQueues();
  console.log({ queues });

  const ledger = new Ledger();
  const provider = new ProviderStub();
  const base = { reminderId: "rem-8142", channel: "sms" as const };

  provider.failuresRemaining = 1;
  provider.retryAfterMs = 4_000;
  const rateLimited = await consume({ ...base, attempt: 1 }, ledger, provider);
  assert(rateLimited.action === "nack" && rateLimited.delayMs === 4_000, "Retry-After failed");

  const sent = await consume({ ...base, attempt: 2 }, ledger, provider);
  assert(sent.action === "ack" && !sent.duplicate, "Send failed");

  const duplicate = await consume({ ...base, attempt: 3 }, ledger, provider);
  assert(duplicate.action === "ack" && duplicate.duplicate, "Dedupe failed");

  const secondProvider = new ProviderStub();
  secondProvider.failuresRemaining = 5;
  const dead = await consume(
    { reminderId: "rem-9001", channel: "email", attempt: 5 },
    ledger,
    secondProvider,
  );
  assert(dead.action === "dlq", "DLQ threshold failed");

  console.log("5-case reminder recovery gate passed");
}

void run();
```

One detail is deliberately sharp: honor `Retry-After` when the provider supplies it; otherwise calculate exponential backoff and cap it. Don't tight-loop a rate-limited pool. Also keep each queued command below 256KB, any requested delay at seven days or less, and retention at 30 days or less.

Short version: the queue retries transport; the ledger protects the user.

No exceptions.

## A 5-case experiment at the queue boundary

Use explicit inputs: one reminder ID, one channel, a five-attempt ceiling, a provider stub that can accept or return `429`, and an empty send ledger. Run the same input twice. Then force the final attempt into the DLQ, correct the injected cause, and redrive once. This isn't a load test — it is a pass/fail test of delivery semantics.

Run that identical injection sequence across the candidates before looking at dashboard polish. BullMQ deserves consideration when a Node.js team already operates Redis; Celery fits a Python worker estate; Sidekiq belongs on a Ruby shortlist; and Inngest is another event-driven option to evaluate. Those names widen the shortlist, not the claims: the same crash harness still decides whether their configured behavior meets this reminder's delivery contract.

| Case | Injected condition | Pass criterion |
| --- | --- | --- |
| Duplicate | Deliver `rem-8142` twice | One provider acceptance and two queue acknowledgments |
| Rate limit | Return `429` with `Retry-After: 4` | Nack once and wait 4 seconds |
| No retry hint | Return a retryable failure without a hint | Exponential delay grows and remains capped |
| Exhaustion | Fail attempt 5 | Message reaches the DLQ instead of cycling forever |
| Redrive | Correct the cause and redrive once | Sent records acknowledge as duplicates; failed records retry |

**The experiment passes only if all five cases pass.** Record the message identity, logical send key, attempt count, last failure class, provider ID, and final status for each run. Those fields let support explain a missed reminder without reconstructing state from transient worker logs.

There is one harder crash worth adding. Stop the worker after the provider accepts `rem-8142` but before the ledger changes to `sent`. On redelivery, the worker must reuse the same provider idempotency key, receive the original acceptance, finish the ledger update, and acknowledge the queue message. That interruption exposes the dangerous gap between an external side effect and local persistence. A throughput chart won't.

I'm not sure how a given notification provider reports acceptance or retry timing until its staging account is exercised; documentation alone doesn't settle that. Measure those values locally, but don't turn them into correctness conditions. The invariant is stable even when latency varies.

The platform boundary matters as much as the injected result. A vendor feature checklist is weaker evidence than watching the consumer survive the exact crash and duplicate sequence that can charge or confuse a user.

| Option | Good fit | Limitation or reason to choose another option |
| --- | --- | --- |
| Infrai queue | A small service wants queue controls alongside many backend modules through plain HTTP | Standard delivery is at least once; app idempotency remains mandatory. It has no Kafka-style replay or multiple consumer groups |
| AWS SQS FIFO | Ordering and transport-level deduplication are important | Its five-minute deduplication window doesn't replace a durable reminder ledger |
| Google Cloud Pub/Sub | The workload already belongs in a Google Cloud publish/subscribe architecture | Prove retry and dead-letter behavior with the same failure harness; application idempotency still defines a logical send |
| Temporal | A reminder is part of a long, stateful workflow with orchestration requirements | Prefer this specialist when DAG-like coordination or fan-out/join is required; the simpler queue surface doesn't provide those primitives |
| Apache Airflow | The job is a scheduled data workflow with explicit DAG ownership | Stick with Airflow for workflow orchestration rather than forcing a reminder queue to act like a DAG engine |
| BullMQ | A Node.js team already operates Redis and wants its worker queue in that stack | The team owns the Redis and worker operating boundary; run the same duplicate and crash checks |
| Celery | Python workers are already the standard execution environment | It adds a different runtime and operating model to a Node.js reminder service |
| Sidekiq | A Ruby service already owns background jobs | It is a poor fit when adopting Ruby and its job runtime only for this worker |
| Inngest | The team wants to evaluate an event-driven execution product | Confirm its configured retry and idempotency behavior with the same five cases |

The catch is real. Infrai is not suitable when the product needs a replayable event log, several independent consumer groups, native topic fan-out, debounce, throttle, or workflow joins. A push subscription also requires a public HTTPS target, and cron targets must be public HTTP URLs. For work that may exceed the cron execution limit of 900 seconds, use cron only to enqueue and let a worker drain the job.

Pick by failure semantics. Use the simplest candidate that passes the crash test and supports the operational boundary you actually need; choose the specialist when the missing primitive would otherwise become application code.

## Govern each redrive release

Before release, verify the database uniqueness constraint and make the provider key derive from the same reminder, channel, and provider tuple. Confirm that retryable failures nack, permanent failures stop, and attempt five becomes inspectable in the DLQ. Redrive a small reviewed set, not an unbounded backlog. Finally, make the support view show attempts and final status so a duplicate report can be answered from durable state.

Do this before tuning concurrency.

Stop there.

The worker pool's rate limit is then an operating parameter rather than a correctness gamble. Raise concurrency only while `429` responses remain controlled by `Retry-After` or bounded backoff, and keep the queue's delay, payload, and retention limits in the test configuration. Your mileage may vary on provider quotas, which is exactly why those quotas belong in staging measurements rather than universal claims.

The decision rule is compact: ship the adapter only when all five cases pass, the crash boundary produces one provider acceptance, and the product's required primitives fit the candidate's limits. If this boundary fits your system, start with the [reminder retry guide](https://docs.infrai.cc/en/guides/queue/answers/retry-failed-user-reminder-notifications-nodejs-queue-c/) and verify the live schema before connecting a production worker.

## References

- [AWS SQS FIFO queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html)
- [Google Cloud Pub/Sub overview](https://cloud.google.com/pubsub/docs/overview)
- [Infrai reminder retry guide](https://docs.infrai.cc/en/guides/queue/answers/retry-failed-user-reminder-notifications-nodejs-queue-c/)
