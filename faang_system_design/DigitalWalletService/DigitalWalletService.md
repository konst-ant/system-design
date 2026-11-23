# Digital Wallet and Account Balance Service — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `DigitalWalletService_component.excalidraw`, `DigitalWalletService_sequence.excalidraw`.*

## 1. Problem & Scope

Design a service that maintains user balances and processes transfers, guaranteeing an account can never go negative under concurrent operations and that every balance is fully auditable down to the transaction that produced it.

**In scope:** the ledger model, atomic balance-affecting operations, hot-account handling.

**Explicitly deferred:** currency conversion, KYC/compliance checks, payment-provider integration for funding/withdrawal (own exercise — see `PaymentProcessingSystem.md` for the external-provider leg).

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~20M active accounts; ~5K transactions/sec at peak.
- **Hot accounts:** some (e.g. merchant settlement accounts) receive thousands of concurrent credits/debits.
- **Latency:** transaction latency p99 < 500 ms.
- **Correctness:** balance reads used for authorization must be strongly consistent; the ledger must reconcile to the cent under concurrent load and retries.

**Capacity check:** 5K TPS is a modest write rate overall, but it's not evenly spread — a single popular merchant account can receive a large fraction of that concurrently, meaning the real design constraint is hot-account contention, not aggregate throughput.

## 3. Interface

```
POST /accounts/{id}/transfer {to_account, amount, idempotency_key} -> {transaction_id, status}
GET /accounts/{id}/balance -> {balance}  // strongly consistent read
```

## 4. High-Level Architecture — `DigitalWalletService_component.excalidraw`

- **Client → Wallet Service**: initiates a transfer or balance check.
- **Wallet Service → Ledger Store**: the source of truth — every movement recorded as a balanced debit/credit pair (§5).
- **Wallet Service → Balance Cache**: a derived, strongly-consistent view kept in lockstep with the ledger for fast authorization reads.
- **Reconciliation Job → Ledger Store**: periodically verifies the cached balance still sums correctly from the underlying ledger entries.

## 5. Deep Dive — Double-Entry Ledger & Atomic Conditional Debit — `DigitalWalletService_sequence.excalidraw`

**Options considered:**
- A single mutable `balance` field per account, updated via `balance = balance - amount` — simple, but records no record of *why* the balance changed (no audit trail), and under concurrency two simultaneous debits can both read a sufficient balance before either writes, both succeed, and drive the account negative when only one should have been allowed.
- **Double-entry ledger + atomic conditional debit — the pick:** every transaction is a balanced debit/credit pair across exactly two accounts (the ledger's core invariant: the sum of all entries is always zero); a debit only commits via one atomic operation that checks sufficient balance and writes the entry pair together, or not at all.

**Terms:**

| Term | Meaning |
|---|---|
| Double-entry ledger | Every money movement recorded as a balanced debit+credit pair across two accounts — the append-only source of truth. |
| Derived balance | An account's balance is the sum of its ledger entries, never a freely-mutable field. |
| Atomic conditional debit | Checks sufficient balance and writes the entry pair in one indivisible operation — no separate read-then-write. |
| Reconciliation | Periodically verifying a cached derived balance still matches what the ledger entries sum to. |

**Worked timeline** (Account A starts at balance 100):

| Time | Event | Outcome |
|---|---|---|
| t=0 | Two concurrent transfer-80 requests arrive for Account A | both check balance via the same atomic path |
| t=0 | Request 1's atomic debit: `balance >= 80`? (100 >= 80, yes) → commits entries, balance now 20 | **succeeds** |
| t=0 | Request 2's atomic debit: `balance >= 80`? (20 >= 80, no — sees the already-updated value) | **rejected cleanly**, account never goes negative |

**Correctness pitfall:** caching "current balance" as an independently-mutable field, updated optimistically and only *eventually* reconciled against the ledger (instead of being strictly derived via the same atomic operation that writes the ledger entries), reopens the exact race condition the ledger exists to prevent — the balance check and the ledger write must happen as one atomic step, the same "no separate read-then-write" lesson as `ECommercePlatform.md` §5 and `RateLimiter.md` §5, here applied to money rather than inventory or request quota.

## 6. Data Model

- Ledger entry: `entry_id, account_id, amount (signed), transaction_id, created_at` — immutable once written.
- Transaction: `transaction_id, debit_account, credit_account, amount, idempotency_key, status`.
- Balance cache: `account_id -> cached_balance, last_reconciled_entry_id` — derived, periodically reconciled, never the sole source of truth.

## 7. Scaling, Failure & Trade-offs

- A hot account (merchant settlement) can be sharded at the application level into sub-ledgers that periodically consolidate, trading a small amount of settlement latency for removing the single-row contention point — the same hot-partition handling pattern as `WebCrawler.md` §7 and `MessageStorageSystem.md` §7, applied to a financial ledger.
- Every transaction carries an idempotency key (same mechanism as `PaymentProcessingSystem.md` §5) so a client retry after a timeout never double-applies a transfer.
- Strong consistency on balance reads is non-negotiable for authorization decisions, but historical transaction history (statements, past entries) can safely serve from a read replica — only the "can this debit proceed right now" path needs the strict guarantee.

## 8. Wrap-up

Delivered: a double-entry ledger with atomic conditional debits that guarantees a balance can never go negative under concurrency, fully auditable by construction. Not covered due to time: currency conversion, KYC/compliance, and the external funding/withdrawal provider integration.