# Rate Limiter — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `RateLimiter_component.excalidraw`, `RateLimiter_sequence.excalidraw`.*

## 1. Problem & Scope

Design a service that decides, on the hot path of every API request, whether a given client/endpoint combination is allowed to proceed, enforcing policies like "100 req/min per API key" or "5 req/sec per IP on `/login`".

**In scope:** multiple limiting algorithms, consistent enforcement across many stateless gateway/app instances, low-latency decisions, graceful behavior under limiter-backend failure.

**Explicitly deferred (would need separate follow-up):** a UI/API for managing policies, long-term analytics on throttled traffic, billing-tier business logic. These don't change the core architecture, so I'd rather spend the time budget on the enforcement path itself.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~50,000 distinct API clients; ~10K RPS average, ~100K RPS peak platform-wide.
- **Latency:** the check sits on every request's critical path → budget p99 < 5 ms for the decision itself.
- **Consistency:** approximate/bounded enforcement is acceptable — a client overshooting its limit by a small, bounded margin (e.g. <5%) under extreme concurrency is fine; we are not building a financial ledger.
- **Availability:** the limiter must not become the reason the whole API goes down. A policy choice (fail-open vs fail-closed) is required per endpoint sensitivity.

**Capacity check:** at 100K RPS peak, a naive "one Redis round-trip per request" design needs the counter store to sustain ~100K ops/sec. A single Redis node comfortably exceeds that, but for headroom and blast-radius control I'd shard counters across ~8–16 Redis nodes by `hash(client_id, endpoint)`, so each shard only sees ~6–12K ops/sec. Memory: 50,000 clients × ~100 endpoints × ~100 bytes/counter ≈ 500 MB — trivial, fits in a small cluster. (The ~100 bytes is mostly Redis per-key overhead, not the data — the actual state is just two small numbers; see §5 for exactly what's stored.)

## 3. Interface

This is an internal cross-cutting concern, not a public REST API — it's a library/sidecar call made by the gateway or service mesh proxy:

```
check(client_id, endpoint, cost=1) -> { allowed: bool, remaining: int, reset_at: timestamp }
```

## 4. High-Level Architecture — `RateLimiter_component.excalidraw`

- **Client → API Gateway**: normal request.
- **Gateway → Rate Limiter module** (embedded library or sidecar): calls `check()` before forwarding.
- **Rate Limiter → Local in-proc cache**: fast path — a short-lived token lease checked out from Redis (see §5).
- **Rate Limiter → Redis cluster**: source of truth counters, sharded by key.
- **Policy/Config service → Rate Limiter**: pushes limit definitions (per client/endpoint); pulled/cached periodically, *not* looked up synchronously per request.
- **Gateway → Backend service**: only on `allowed = true`.

## 5. Deep Dive — Algorithm Selection & Token Bucket Mechanics — `RateLimiter_sequence.excalidraw`

**Algorithm options considered:**
- Sliding Window Log — exact, but O(n) memory per key (stores every timestamp) — too expensive at this scale.
- Sliding Window Counter — O(1) memory, bounded small error from edge-weighting — a solid alternative.
- Leaky Bucket — shapes traffic to a constant output rate; the wrong fit here, since the goal is to let bursts through and bound the average, not force a constant trickle.
- **Token Bucket — the pick:** O(1) memory, and burst tolerance is an explicit, independently-tunable parameter (bucket capacity `B`, separate from the steady rate) rather than an incidental edge effect of window boundaries — the natural fit for "allow bursts, enforce an average," and the model that cleanly supports the lease-based hot-path optimization below.

**Terms:**

| Term | Meaning |
|---|---|
| Global counter (Redis) | One record per key: `tokens_remaining`, `last_refill_time`. Single source of truth across all instances. |
| Rate | Steady allowed throughput, from policy (e.g. 1000/100s → 10/s). |
| Bucket capacity `B` | Max burst size — an independent design choice, not derived from rate. |
| Initialization | First-ever or post-expiry access → `tokens_remaining = B`. One-time, not refill. |
| Refill | Every access after init: add `elapsed × rate`, capped at `B`. |
| Local cache (lease) | Per-instance pool checked out from the global counter — a withdrawal, not a mirror. Avoids a Redis call per request. |
| Lease size | Tokens granted per checkout — small, fixed, independent of `B`. |
| Local cache lifetime | How long an instance spends its lease before checking in again — its own short clock, unrelated to Redis-side key expiry. |
| Discard on expiry | Unspent lease tokens are dropped, not returned — only ever makes enforcement more conservative. |

**Worked timeline** (`rate=10/s`, `B=50`, `lease=5`, local cache lifetime `=1s`):

| Time | Event | Redis (`tokens_remaining`) | Local cache |
|---|---|---|---|
| t=0 | Key never seen → **initialize** to `B=50`. Instance A checks out a lease of 5 | 50 → 45 | A: 5 |
| t=0–1s | A serves 3 requests from its lease, no Redis calls | 45 | A: 5→2 |
| t=0.5 | Instance B's first lease: **refill** `0.5×10=5` → 50 (hits cap). Checkout 5 | 50 → 45 | B: 5 |
| t=1.0 | A's lease lifetime ends, 2 unspent tokens **discarded**. New lease: **refill** `0.5×10=5` → 50 (cap). Checkout 5 | 50 → 45 | A: 5 |

**Correctness pitfall:** the checkout above must be a single atomic step. Doing it as two separate Redis round-trips (read `tokens_remaining`, then write the new value) is a race condition — two concurrent requests from different gateway instances can both read the same pre-decrement value and both proceed, overshooting the limit. Fix: push the whole read-refill-checkout sequence into one **Redis Lua script** (`EVAL`), executed atomically, using Redis's own clock for the elapsed-time calculation — never trust client wall clocks across instances, since clock skew would corrupt the refill math.

## 6. Data Model

- Redis key: `rl:{client_id}:{endpoint}` → `{tokens_remaining, last_refill_time}`. TTL set to a few multiples of the time-to-refill (housekeeping only, to reclaim memory from inactive clients — unrelated to the local cache's own short lifetime; see §5).
- Policy store (small KV/relational table): `client_id, endpoint_pattern, rate, bucket_capacity`.

## 7. Failure Handling & Trade-offs

- **Redis shard down:** that shard's keys are briefly unavailable. Policy-driven choice: fail-open for low-risk endpoints (availability > strict enforcement), fail-closed for sensitive ones (e.g. login, payment).
- **Hot key** (one client hammering a shard): the local lease checkout (§5) sheds most of that load before it reaches Redis.
- **Global vs regional limits:** this design assumes a single-region Redis cluster. True multi-region enforcement would need either per-region sub-limits (split the global limit N ways) or cross-region counter replication — flagged as a follow-up, not solved here given the time box.

## 8. Wrap-up

Delivered: a sharded, atomic, low-latency limiter with a clear availability/accuracy trade-off and a realistic hot-path optimization. Not covered due to time: multi-region limits, a self-serve policy-management UI, and detailed abuse/analytics tooling.
