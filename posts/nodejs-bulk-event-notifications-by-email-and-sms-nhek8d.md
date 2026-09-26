# Nodejs Bulk Event Notifications by Email and SMS (Queue Worker Evidence)

Short answer: put each marketplace order event into a durable outbox, let a worker claim a bounded batch, and record a separate, append-only attempt for every email or SMS submission. Run a scheduled poller against stale or pending work as a recovery path. The trade-off is deliberate: more storage and bookkeeping buys evidence that explains what the system intended, what it submitted, and what a channel later reported. For a seller notification, that audit trail matters more than removing one database write.

The flow is plain. Checkout commits the order and notification intent together. A worker claims intents, resolves the seller's eligible channels, renders immutable content, submits channel-specific batches, and stores each result. A cron-triggered poller calls the same worker path; it does not contain a second delivery implementation.

## How should Nodejs send bulk event notifications by email?

Separate four facts that are often collapsed into one `sent` boolean: the order event existed, policy selected a channel, an adapter accepted a submission, and a later channel event reported an outcome. Those facts happen at different times. They need different timestamps and stable identifiers.

An acceptance response is evidence of submission, not proof that a person received or read a message. Keep that distinction in the schema and in operational language. It stops a compliance export from making a stronger claim than its records support. For email, retain the rendered body hash, recipient, sender domain, template revision, submission time, adapter message identifier, and later disposition. DKIM lets a receiving system validate a signing domain's responsibility for a message and whether signed content changed in transit. Store the selector and signing domain used for the attempt, but never reinterpret a valid signature as proof of inbox placement. SMS has a less obvious evidence field: encoding. A single message can carry 160 GSM-7 characters or 70 UCS-2 characters. Concatenated segments have lower per-segment limits: 153 and 67 respectively. One curly quote can change segmentation. Record the rendered text, detected encoding, and estimated segment count before submission; exact billing remains a downstream fact, not an estimate disguised as one.

Evidence first.

## A focused TypeScript worker

This example keeps infrastructure behind interfaces. Claims must be atomic, evidence is written per attempt, and continuous processing and scheduled recovery both call `drain`. A store can use row locking or an atomic lease update, provided two workers cannot own the same intent at once.

```ts
type Channel = "email" | "sms";
type Intent = {
  id: string;
  orderId: string;
  channel: Channel;
  destination: string;
  templateRevision: string;
};
type Submission = {
  intentId: string;
  idempotencyKey: string;
  destination: string;
  content: string;
  contentSha256: string;
  templateRevision: string;
  smsEncoding?: "GSM-7" | "UCS-2";
  smsSegments?: number;
};
type SubmitResult = {
  intentId: string;
  accepted: boolean;
  externalId?: string;
  reasonCode?: string;
  submittedAt: Date;
};

interface IntentStore {
  claim(limit: number, leaseUntil: Date): Promise<Intent[]>;
  appendAttempts(results: SubmitResult[]): Promise<void>;
  complete(ids: string[]): Promise<void>;
  release(ids: string[], availableAt: Date): Promise<void>;
}
interface ChannelAdapter {
  submit(batch: Submission[]): Promise<SubmitResult[]>;
}
type Dependencies = {
  store: IntentStore;
  adapters: Record<Channel, ChannelAdapter>;
  render(intent: Intent): Promise<Submission>;
  now(): Date;
};

export async function drain(deps: Dependencies, limit = 100): Promise<number> {
  const now = deps.now();
  const intents = await deps.store.claim(
    limit,
    new Date(now.getTime() + 60_000),
  );
  if (intents.length === 0) return 0;

  const submissions = await Promise.all(intents.map(deps.render));
  const batches: Record<Channel, Submission[]> = { email: [], sms: [] };
  for (const submission of submissions) {
    const intent = intents.find((item) => item.id === submission.intentId);
    if (!intent) throw new Error(`Missing intent ${submission.intentId}`);
    batches[intent.channel].push(submission);
  }

  const results: SubmitResult[] = [];
  for (const channel of ["email", "sms"] as const) {
    if (batches[channel].length > 0) {
      results.push(...await deps.adapters[channel].submit(batches[channel]));
    }
  }

  await deps.store.appendAttempts(results);
  await deps.store.complete(
    results.filter((result) => result.accepted).map((result) => result.intentId),
  );
  await deps.store.release(
    results.filter((result) => !result.accepted).map((result) => result.intentId),
    new Date(now.getTime() + 30_000),
  );
  return intents.length;
}

export async function scheduledRecovery(deps: Dependencies): Promise<void> {
  while (await drain(deps, 100) > 0) {
    // Continue through work that is currently available.
  }
}
```

The sample does not infer retryability from a free-form error string. Each adapter should map documented outcomes into a small internal taxonomy: accepted, retryable, permanently rejected, or suppressed. Unknown outcomes need review or a conservative policy. Blind retries can create duplicates.

The idempotency key belongs to an intent, not a batch. Batch membership changes as workers retry and leases expire; the logical notification does not. Persist the key across attempts, and enforce uniqueness in the store rather than an in-memory set.

## Evidence before throughput

A useful attempt record is boring and explicit. It connects `orderId -> intentId -> attemptId -> externalId`, with timestamps from the application clock and later downstream timestamps. It records the policy revision that chose email, SMS, both, or neither. Without that revision, an investigator can see what was submitted but cannot reconstruct why.

Keep sensitive payloads under a retention policy appropriate to the marketplace's obligations. A content hash can show that two stored representations match, but it cannot recreate deleted content or establish that the original content was lawful. Hashes, payloads, consent evidence, suppression decisions, and transport responses answer different questions.

**Do not let observability become a second, weaker audit database.** Metrics can aggregate queue age, claim counts, acceptance categories, retries, and terminal failures. Logs can carry correlation identifiers and reason codes. Authoritative evidence belongs in controlled records with defined access and retention; full addresses, phone numbers, and bodies do not belong in routine logs.

Backpressure is policy. Bound every claim and adapter batch, cap concurrent submissions, and stop claiming when a channel is unhealthy. Notification delay may rise, but the database stays protected and retry order remains understandable. For a small team, predictable degradation is easier to operate than a worker that consumes an entire backlog at once.

## Failure paths that deserve tests

The happy path proves little. Test a process exit after downstream acceptance but before `appendAttempts`, because at-least-once execution can then submit twice. A stable idempotency key is the primary defense when a downstream transport honors it. Otherwise, reconciliation must compare stored attempts and downstream events before another submission. A lease expiring is not a reason to mark an intent complete. Then test mixed batch results. One invalid destination must not convert 99 accepted submissions into 100 retries. Persist item-level outcomes before changing intent state. If an adapter returns an incomplete result set, retain unmatched intents for reconciliation and flag the contract violation.

Other permanent fixtures should cover an order transaction that rolls back, two workers claiming at once, a seller becoming suppressed between intent creation and rendering, Unicode changing SMS segmentation, a late outcome arriving after a retry, and the poller overlapping the continuous worker. A fixed clock and deterministic identifiers make the evidence chain exactly assertable.

Tiny tests pay rent.

Deploy just as deliberately. Add evidence tables before producers, ship the worker disabled, exercise it with non-delivering adapters, and raise concurrency gradually. Watch the age of the oldest available intent, not raw queue length alone. A busy marketplace can have a large healthy queue while one permanently leased record is old and invisible in a count.

## The operating rule

Before enabling seller order alerts, verify with compliance and operations that an order and its intent commit together; channel eligibility comes from a versioned policy; content and SMS encoding are captured before submission; each logical notification has a stable idempotency key; item-level outcomes survive partial batches; leases expire safely; scheduled polling reuses the worker path; and reconciliation attaches late outcomes without erasing earlier attempts. Confirm that dashboards describe submission and disposition accurately, never as human receipt. Finally, run rollback, overlap, crash, and Unicode cases against the deployed schema.

The design is ready when durable records can answer a support or compliance question without application logs: which order triggered the notice, why the channel was eligible, what representation was submitted, which attempt carried it, and what outcome was observed. Throughput tuning can follow measured backlog behavior. Preserve the chain first.

## Sources

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- Twilio, SMS character limits and segmentation (GSM-7/UCS-2): https://www.twilio.com/docs/glossary/what-sms-character-limit
