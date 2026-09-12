# Autonomous Agent Spend Governance: 3 Audit Gates for Account Budget Limits

## Short answer

Short answer: enforce the budget in a server-side account ledger before every model or tool call, then repeat the check after the call and at the loop boundary. A Node.js or Python agent should treat the ledger as the authority; prompts, SDK callbacks, and UI warnings are useful signals, not spending controls.

For a B2B SaaS access review, this matters because the output has to be signed by a customer, not merely generated. The review run needs a trace of who approved each action, what it cost, and why the loop stopped. An account cap gives you a hard outer wall. A per-run reservation and a per-step allowance make that wall explainable.

## Where should an autonomous agent enforce budget limits in Node.js and Python?

Put enforcement at three audit gates: reservation before a request, settlement after the provider response, and a loop guard before the next action. The same protocol works from Node.js, Python, or a worker written in another language because the state lives behind an ordinary service boundary.

The reservation is deliberately pessimistic. Estimate input tokens, the maximum output you permit, and a fixed allowance for tool calls. Hold that amount against the account before sending anything. Settlement releases the unused hold or records the actual charge. The loop guard checks both the account ceiling and the run ceiling, so a single run cannot consume all of a busy tenant's allowance.

Here is a compact TypeScript implementation. The ledger methods are generic HTTP-backed interfaces; keep provider-specific pricing and authentication behind the adapter.

```ts
type Budget = {
  accountCents: number;
  runCents: number;
};

type Reservation = {
  id: string;
  accountId: string;
  runId: string;
  heldCents: number;
};

interface Ledger {
  reserve(input: { accountId: string; runId: string; amountCents: number }): Promise<Reservation | null>;
  settle(input: { reservationId: string; actualCents: number }): Promise<void>;
  spent(input: { accountId: string; runId: string }): Promise<Budget>;
}

function estimateCents(inputTokens: number, outputTokens: number, toolCalls: number): number {
  const tokenAllowance = Math.ceil((inputTokens + outputTokens) / 1000) * 3;
  return tokenAllowance + toolCalls * 2;
}

export async function runAccessReview(
  ledger: Ledger,
  accountId: string,
  runId: string,
  steps: number,
): Promise<void> {
  for (let step = 0; step < steps; step += 1) {
    const before = await ledger.spent({ accountId, runId });
    if (before.accountCents >= 500 || before.runCents >= 80) {
      throw new Error("budget_exhausted");
    }

    const reservation = await ledger.reserve({
      accountId,
      runId,
      amountCents: estimateCents(2_000, 1_000, 1),
    });
    if (!reservation) throw new Error("reservation_denied");

    const started = Date.now();
    try {
      const result = await callModelAndTools();
      const actualCents = Math.max(1, Math.ceil(result.usage.totalTokens / 1000) * 3);
      await ledger.settle({ reservationId: reservation.id, actualCents });
      await appendAuditEvent({ accountId, runId, step, actualCents, latencyMs: Date.now() - started });
    } catch (error) {
      await ledger.settle({ reservationId: reservation.id, actualCents: 0 });
      throw error;
    }
  }
}

async function callModelAndTools(): Promise<{ usage: { totalTokens: number } }> {
  return { usage: { totalTokens: 1_000 } };
}

async function appendAuditEvent(event: Record<string, unknown>): Promise<void> {
  void event;
}
```

The important detail is atomicity. `reserve` must compare the account's committed spend plus outstanding holds with the cap in one transaction. A read followed by a separate write lets two workers both pass the check. Picture two access-review workers waking at the same millisecond: each reads 78 cents on an 80-cent run cap, each sees room for a two-cent reservation, and both send a request before either write is visible. The ledger must serialize that decision, include outstanding holds in the comparison, and return one denial. Give every reservation an idempotency key such as `accountId/runId/step`; retries then settle the same hold instead of creating a second charge. Record the transaction id in the audit event so a reviewer can see which decision won.

In Python, use the same ledger contract and put the check in the worker middleware. Do not rely on a decorator around one SDK call if tools can invoke another model, queue, or browser worker. The boundary must surround the whole action graph.

## What does an auditable budget ledger need to record?

An access review is a small financial system. Store the account id, run id, actor or service identity, reservation id, model class, token counts, tool name, estimated amount, settled amount, currency, timestamps, and the decision (`allowed`, `denied`, or `stopped`). Keep the request hash rather than raw secrets or sensitive document contents. OWASP's Secrets Management Cheat Sheet recommends controlling secret exposure and rotation; that guidance also supports keeping credentials out of audit payloads.

Separate authorization from accounting. A policy service can answer “may this run read directory data?” while the ledger answers “may it spend another 12 cents?” Joining both decisions in one immutable event gives a reviewer a defensible chain: policy version, budget snapshot, action, result.

Short events help.

Hard stop.

A practical retention policy keeps detailed step events for the review window and rolls up account totals for longer reporting. Use UTC timestamps and a monotonic sequence per run. If events arrive twice, deduplicate by reservation id, not by timestamp. A delayed settlement must not reopen a run that the loop guard already stopped.

## Failure modes that make caps look enforced

The first trap is checking only at the start of a loop. A tool can fan out into five calls after that check. Count the fan-out in the reservation, or make each child action reserve independently. The second trap is trusting provider usage reported after a timeout. Mark the reservation as pending and reconcile it asynchronously; never assume a timeout means zero cost.

The third trap is placing a cap in the prompt. Models can follow “stop at 80 cents” until a tool result changes the plan. The prompt can explain the remaining allowance, but only the ledger can deny the next request.

I also keep a separate kill switch for a runaway run. It is operational, not a pricing rule: a maximum wall-clock duration, maximum tool depth, and maximum number of parallel branches. These limits catch loops whose token estimate is small but whose side effects are expensive.

Your mileage may vary on the estimate. Provider tokenization, cached input, and tool billing differ, so tune the estimator from settled events and expose the error margin in the audit record.

## Choosing the boundary for a small team

For one worker and one provider, a transactional table with a unique reservation key is usually enough. Add a queue-backed ledger when several workers share an account or when settlements can arrive out of order. A distributed in-memory counter is fast, but it needs durable reconciliation before it can support a signed access review.

The catch is that a hard account cap can reject a legitimate review halfway through. That is the right behavior for compliance, but it is a poor fit for exploratory analytics where partial results are acceptable. Use a soft warning and a larger run allowance for exploration; use hard reservations and a short maximum depth for actions that change permissions. Stick with a simpler per-run cap when your product cannot explain account-level attribution yet.

Keep the policy visible in the review UI: cap, current committed spend, outstanding holds, stop reason, and the identity that changed the cap. Do not hide a budget decision inside a generic “agent failed” status.

Before shipping, exercise the ledger with concurrent reservations, retry the same step, inject a provider timeout, and replay a duplicated settlement. Verify that the account total never exceeds its cap, that every denied action has an event, and that a reviewer can reconstruct the final recommendation without reading application logs. That checklist is more valuable than a clever agent prompt.

## References

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- W3C Trace Context Recommendation: https://www.w3.org/TR/trace-context/

## Further reading

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- W3C Trace Context Recommendation: https://www.w3.org/TR/trace-context/
