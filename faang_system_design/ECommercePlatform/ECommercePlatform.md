# E-Commerce Platform — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `ECommercePlatform_component.excalidraw`, `ECommercePlatform_sequence.excalidraw`.*

## 1. Problem & Scope

Design an online shopping platform covering product browsing, cart, checkout, and order placement, that never oversells inventory even under concurrent demand spikes.

**In scope:** cart/checkout flow, and — as the deep dive — inventory consistency under concurrency.

**Explicitly deferred:** product search/catalogue (own exercise — see `SearchEngine.md`/`ECommerce` catalog exercise), payment provider integration details (own exercise — see the Payment Processing exercise), fulfillment/shipping logistics. None change the checkout/inventory core.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~50M DAU; ~1M orders/day; seasonal peaks (e.g. Black Friday) at 10x normal traffic.
- **Latency:** checkout p99 < 1 s.
- **Correctness:** inventory must never go negative — overselling is a hard failure, not a tunable trade-off.
- **Availability:** the platform should degrade to read-only browsing rather than fail completely if a downstream service (pricing, recommendations) is unavailable.

**Capacity check:** at 10x peak, a single popular SKU during a flash sale can see thousands of concurrent checkout attempts against a stock count that might only be in the hundreds — this concentrated contention on one row, not the platform's aggregate throughput, is the actual hard part of this problem.

## 3. Interface

```
POST /cart/{sku}/reserve {quantity} -> {reservation_id, expires_at}  // holds stock
POST /checkout {reservation_id, payment_info} -> {order_id}          // confirms the hold
```

## 4. High-Level Architecture — `ECommercePlatform_component.excalidraw`

- **Client → Cart Service**: add to cart → calls Inventory Service to place a short-lived reservation (§5).
- **Client → Checkout Service**: on confirm, finalizes the reservation and hands off to Payment Service.
- **Inventory Service → Inventory Store**: the authoritative available/reserved/sold split (§5).
- **Reservation Expiry Worker**: releases holds that were never confirmed within the TTL.
- **Order Service → Order Store**: durable record created only after a reservation is successfully finalized.

## 5. Deep Dive — Inventory Consistency at Checkout — `ECommercePlatform_sequence.excalidraw`

**Options considered:**
- Decrement inventory directly at checkout with `UPDATE stock SET available = available - 1 WHERE sku = X AND available > 0` as two separate steps (check, then write) — the check-then-write split is a race condition: two concurrent checkouts can both read `available > 0` before either writes, and both proceed, overselling the last unit.
- **Reserve-then-confirm with an atomic, conditional decrement — the pick:** adding to cart atomically decrements `available` (only if `available > 0`, in one operation) and creates a time-boxed reservation; checkout finalizes an existing reservation rather than touching inventory again; an unconfirmed reservation auto-releases after its TTL.

**Terms:**

| Term | Meaning |
|---|---|
| Available | Stock not currently reserved or sold — the only number a new reservation can draw from. |
| Reserved | Stock held by an in-progress checkout, time-boxed by a TTL. |
| Sold | Stock permanently committed to a completed order. |
| Atomic conditional decrement | `available -= 1` only if `available > 0`, executed as one indivisible operation (e.g. Redis `DECR` guarded by a Lua script, or a DB conditional `UPDATE`). |
| Reservation TTL | How long a hold lasts before auto-releasing back to `available` if checkout never completes. |

**Worked timeline** (SKU starts with `available = 3`):

| Time | Event | Available | Reserved | Sold |
|---|---|---|---|---|
| t=0 | User A reserves 1 (atomic decrement succeeds) | 2 | 1 | 0 |
| t=0 | User B reserves 1 (atomic decrement succeeds) | 1 | 2 | 0 |
| t=0 | User C reserves the last 1 (atomic decrement succeeds) | 0 | 3 | 0 |
| t=0 | User D attempts to reserve 1 → `available = 0` → rejected immediately | 0 | 3 | 0 |
| t=10min | User A's reservation expires unconfirmed → released back | 1 | 2 | 0 |
| t=10min+ε | User C's payment succeeds → reservation finalized | 1 | 1 | 1 |

**Correctness pitfall:** the check-and-decrement for a reservation must be a single atomic operation — this is the exact same race condition already covered for rate-limit counters (`RateLimiter.md` §5) and social-media like counts (`SocialMediaPlatform.md` §5): two concurrent "reserve" calls must never both read `available = 1` and both proceed. The fix is identical in shape: one atomic conditional operation, never a separate read followed by a separate write.

## 6. Data Model

- Inventory: `sku, available, reserved, sold` (three-way split, not a single number).
- Reservation: `reservation_id, sku, quantity, user_id, created_at, expires_at, status`.
- Order: `order_id, user_id, items, total, created_at` — written only on successful checkout finalization.

## 7. Scaling, Failure & Trade-offs

- The reservation TTL (e.g. 10 minutes) bounds how long an abandoned cart holds real inventory hostage — short enough to free stock quickly, long enough to cover a normal checkout flow.
- If the Payment Service is slow or down, the reservation simply expires and releases — no inventory is lost, the user just has to retry.
- Pricing/recommendation service outages degrade the browsing experience but must not block checkout for items already in a confirmed cart — checkout only depends on Inventory, Payment, and Order services.

## 8. Wrap-up

Delivered: a reserve-then-confirm inventory model with atomic conditional decrements that guarantee zero overselling under concurrency. Not covered due to time: catalogue/search, payment provider integration details, and fulfillment logistics.