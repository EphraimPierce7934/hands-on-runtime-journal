# Node.js Message Size: Publish Per Waiter or the Whole Queue

Short answer: publish a compact delta for each changed waiter when the client can be trusted to maintain ordered state; publish a whole-queue snapshot when recovery simplicity matters more than repeated bytes. For a small SaaS customer queue display, serialize both shapes with production-like names, optional fields, and queue lengths, then compare total bytes across reconnects as well as steady-state updates.

The simple approach is attractive: every change sends the current queue, so any accepted message repairs the display. It also repeats unchanged waiters. A per-waiter message avoids that repetition, but it turns ordering, deletion, and recovery into protocol responsibilities. The useful comparison is bytes plus trust, not bytes in isolation.

## Should Node.js publish per waiter or send the whole queue message?

Assume the display needs a stable waiter ID, position, status, and update time. A full snapshot wraps every visible waiter in one message. A delta identifies one mutation and includes a monotonically increasing queue revision. These are application-level shapes, not WebRTC requirements.

For a queue of `N` waiters, let `S` be the serialized size of the snapshot envelope plus all `N` records. Let `D` be one serialized delta, and let `C` be the number of changes during the comparison window. The steady-state payload comparison is `S * C` versus the sum of those `C` delta sizes. Compression, transport framing, retransmission, and batching can change bytes on the wire, so measure at both the serializer and transport boundary before treating the result as capacity planning.

A whole snapshot can still be the smaller operational choice when queues are short, changes arrive in bursts that can be coalesced, or clients reconnect frequently. Per-waiter deltas become more compelling as unchanged queue records dominate each snapshot. No universal crossover point follows from the available evidence; your field distribution determines it.

Measure both.

## Measure the payloads in Node.js

This TypeScript script compares UTF-8 JSON payload bytes for one hypothetical 12-waiter queue and four changes. The numbers are experiment inputs, not a benchmark claim. Replace them before making a production decision.

```ts
type WaiterStatus = "waiting" | "called" | "seated";
type Waiter = { id: string; position: number; status: WaiterStatus; updatedAt: string };
type Snapshot = { type: "queue.snapshot"; queueId: string; revision: number; waiters: Waiter[] };
type Delta = { type: "waiter.upsert"; queueId: string; revision: number; waiter: Waiter };

const byteLength = (value: unknown): number =>
  Buffer.byteLength(JSON.stringify(value), "utf8");

const waiters: Waiter[] = Array.from({ length: 12 }, (_, index) => ({
  id: `waiter-${String(index + 1).padStart(3, "0")}`,
  position: index + 1,
  status: "waiting",
  updatedAt: "2026-09-18T10:00:00.000Z",
}));

const snapshot = (revision: number): Snapshot => ({
  type: "queue.snapshot", queueId: "queue-demo", revision, waiters,
});
const delta = (revision: number, waiter: Waiter): Delta => ({
  type: "waiter.upsert", queueId: "queue-demo", revision, waiter,
});

const changed = waiters.slice(0, 4);
const snapshotBytes = changed.reduce(
  (total, _, index) => total + byteLength(snapshot(index + 1)), 0,
);
const deltaBytes = changed.reduce(
  (total, waiter, index) => total + byteLength(delta(index + 1, waiter)), 0,
);

console.log({ snapshotBytes, deltaBytes });
```

Run this against several distributions: the median queue, the upper tail, a burst of calls, and a reconnect. Preserve the serialized fixtures in tests so a later schema change cannot quietly double the payload. One extra display field repeated across every waiter may matter more than the choice of transport library.

Count messages too. A delta per waiter can produce more sends than a coalesced snapshot, even when its payload total is lower. If several waiters change within one display refresh interval, test a small batch of deltas as a third candidate rather than forcing a binary choice.

## Token scope decides how much to trust the client

The publishing unit and authorization unit should be designed together. Give a display token access to one queue, not to every queue owned by the account, and validate that scope when the realtime session is established and resumed.

A queue identifier is not proof of authorization.

With snapshots, the client can replace local state after checking the queue ID and revision. With deltas, it must reject stale revisions, detect gaps, apply each operation once, and request a fresh snapshot when continuity is uncertain. Keep mutation authority off a read-only display token. Typing indicators can usually expire rather than replay; read receipts and waiter status changes need explicit identity and ordering because they alter durable user expectations.

This is the trust trade-off: deltas save repeated data by asking the client to do more correct work. A client that silently applies revision 44 after revision 46 is costly once the public display disagrees with the source of truth.

## Failure and recovery belong in the size comparison

A fair experiment includes recovery. Disconnect the client between two revisions, deliver duplicates, reverse two messages, and remove a waiter while the display is offline. Expected delta behavior is deterministic: duplicates do not change state, a gap triggers resynchronization, and a deletion uses an explicit operation rather than absence. Expected snapshot behavior is simpler: a newer complete revision replaces the older view.

WebRTC specifies peer-to-peer data channels and discusses properties including ordering and reliability. Those transport choices do not define an application queue protocol. If a data channel permits missing or reordered application data, the revision-and-resync rule remains necessary. The same rule is useful over other realtime transports because reconnect boundaries can interrupt an otherwise ordered session.

Do not infer health from average payload size alone. Track serialized payload bytes by event type, messages per queue change, snapshot resync count, revision-gap count, duplicate count, reconnect duration, and time from source mutation to rendered display. Watch the tail.

A calm average can hide one large queue that repeatedly republishes thousands of unchanged fields.

Choose full snapshots first when the bounded queue is small, correctness must survive a minimally trusted client, and resynchronization is common. Choose per-waiter deltas when measured snapshot repetition is material and the client can enforce revisions, idempotency, explicit deletion, and snapshot recovery. A hybrid is often the clean protocol boundary: scoped deltas during a healthy session, plus a canonical snapshot on initial connection and after any detected gap.

Ship the measurement before the optimization. Record byte totals and recovery behavior for both candidates under the same queue fixtures, without including user-entered text or identifiers in telemetry. Then make the choice at a documented queue-size and change-rate boundary. Re-run the fixture whenever the schema grows.

The result to copy is not a fixed threshold. It is the method: compare serialized bytes, include message count and recovery traffic, constrain every token to one queue, and refuse to trust a delta stream that cannot prove continuity.

## References

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
