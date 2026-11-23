# Payment Processing System — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `PaymentProcessingSystem_component.excalidraw`, `PaymentProcessingSystem_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform that accepts payment requests and coordinates their processing through external, unreliable payment providers — guaranteeing every transaction is accounted for exactly once, even when a provider call times out and its true outcome is unknown.

**In scope:** idempotent payment submission, the state machine for an in-flight payment, reconciliation of uncertain outcomes.

**Explicitly deferred:** fraud detection, multi-provider routing/failover policy, settlement/payout batching. `ECommercePlatform.md`'s checkout flow calls into this system for the actual charge.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~10K transactions/sec at peak.
- **Latency:** external provider latency p99 1–3 s, with occasional multi-second timeouts; end-to-end authorization decision p99 < 2 s.
- **Correctness:** 100% financial correctness — no double-charges, no silently lost transactions — and full auditability of every state transition.

**Capacity check:** at 10K TPS with provider calls taking 1–3s, a naive synchronous "one request, one wait" model needs tens of thousands of concurrent in-flight calls just to keep up — routine for an async architecture, but it means a meaningful fraction of in-flight payments will be mid-call at any given moment, which is exactly the population a timeout can strike.

## 3. Interface

```
POST /payments {idempotency_key, amount, source} -> {payment_id, status: pending|succeeded|failed|uncertain}
GET /payments/{id} -> {status, ...}
```

## 4. High-Level Architecture — `PaymentProcessingSystem_component.excalidraw`

- **Client (e.g. `ECommercePlatform.md`'s Checkout Service) → Payment Service**: submits a payment with a caller-chosen idempotency key.
- **Payment Service → Payment Store**: durably records the payment's state machine before calling any external provider.
- **Payment Service → External Provider**: the actual charge attempt — latency-variable, occasionally times out.
- **Reconciliation Job → Payment Store + External Provider**: periodically resolves every payment left in `uncertain` state (§5).

## 5. Deep Dive — Idempotency Keys & Uncertain-State Reconciliation — `PaymentProcessingSystem_sequence.excalidraw`

**Options considered:**
- Treat a provider timeout as a definite failure and immediately retry with a fresh request — simple, but if the original request actually succeeded at the provider (only the response was lost in transit), the retry creates a **second real charge** — a direct double-charge, a hard correctness violation.
- **Idempotency-keyed requests + an explicit `uncertain` state with reconciliation — the pick:** every payment attempt carries a caller-chosen idempotency key that the provider deduplicates on; on timeout, the payment moves to `uncertain` (never retried blindly), and a reconciliation job queries the provider's own "status by idempotency key" lookup to resolve the true outcome before any retry is allowed.

**Terms:**

| Term | Meaning |
|---|---|
| Idempotency key | Caller-generated unique ID per logical payment attempt; the provider uses it to deduplicate retries under the same key. |
| Uncertain state | A timed-out payment whose true outcome isn't yet known — explicitly distinct from `failed`. |
| Reconciliation job | Periodically queries the provider for the true status of every payment still in `uncertain`. |
| State machine | `pending -> {succeeded, failed, uncertain} -> resolved` — every transition durably recorded. |

**Worked timeline** (payment P, idempotency key K):

| Time | Event | State |
|---|---|---|
| t=0 | P submitted to provider with key K | `pending` |
| t=2s | Provider call times out — response never received | `uncertain` (not `failed`, not retried) |
| t=10s | Reconciliation job queries provider: "status of key K?" | provider reports: charge actually succeeded |
| t=10s | P resolved | `succeeded` — no duplicate charge ever created |

**Correctness pitfall:** retrying a timed-out payment under a **new** idempotency key (instead of reusing the original, or waiting for reconciliation) defeats the entire mechanism — if the original attempt actually succeeded under the old key, a retry under a fresh key is indistinguishable, to the provider, from a genuinely new payment, and **will** double-charge the customer. The idempotency key must be chosen once per logical attempt by the caller and never regenerated on retry.

## 6. Data Model

- Payment: `payment_id, idempotency_key, amount, status, provider_ref, created_at, resolved_at`.
- State transition log: append-only, every change to a payment's status recorded with a reason — the audit trail the NFR requires.
- Uncertain queue: payments currently in `uncertain`, polled by the reconciliation job until resolved.

## 7. Scaling, Failure & Trade-offs

- The reconciliation job's query rate is bounded by how many payments are genuinely `uncertain` at once — a small fraction of total volume under normal provider health, growing only during a provider-side incident, which is exactly when it matters most.
- Idempotency keys are also what make `ECommercePlatform.md`'s checkout retries safe end-to-end: if the client's own request to this service times out and retries, resubmitting with the same key is always safe, never a double-submission.
- A provider that's persistently slow (not just occasionally) degrades the uncertain-queue size, not correctness — the system stays correct under provider degradation, just slower to resolve, which is the right trade-off for a hard-correctness domain.

## 8. Wrap-up

Delivered: an idempotency-keyed state machine that turns "the provider call timed out" from an ambiguous risk into an explicit, reconciled state, with a named failure mode (key regeneration on retry) that would otherwise silently reintroduce double-charging. Not covered due to time: fraud detection, multi-provider routing, and settlement batching.