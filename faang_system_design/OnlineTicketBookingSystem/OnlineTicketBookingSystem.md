# Online Ticket Booking System — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `OnlineTicketBookingSystem_component.excalidraw`, `OnlineTicketBookingSystem_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform where users search events and buy tickets, surviving a flash-sale arrival pattern where far more users show up in the first minute than there are seats, without overselling and without the sheer concurrency collapsing the booking system itself.

**In scope:** admission control for extreme concurrent demand, seat-level reservation.

**Explicitly deferred:** event/venue search, pricing/dynamic pricing, refunds/resale. The seat-reservation mechanism reuses `ECommercePlatform.md` §5's atomic conditional decrement, applied per-seat.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** flash-sale scenario — ~500K concurrent users competing for ~50,000 seats within the first minute of sale opening.
- **Latency:** seat-selection response p99 < 1 s.
- **Reservation:** hold TTL 5–10 minutes.
- **Correctness:** zero overselling despite the concurrency spike, even if that means queuing users before they reach checkout.

**Capacity check:** 500K concurrent requests against a 50,000-seat inventory means 90% of demand is mathematically destined to fail — the real design problem isn't the reservation logic itself (that's a solved problem, per `ECommercePlatform.md` §5), it's that letting all 500K hit that logic simultaneously wastes the system's capacity processing near-certain failures instead of serving the users who could actually succeed.

## 3. Interface

```
POST /queue/join {event_id} -> {queue_token, estimated_wait}
POST /seats/{seat_id}/reserve {admission_token} -> {reservation_id, expires_at}
POST /checkout {reservation_id, payment_info} -> {ticket_id}
```

## 4. High-Level Architecture — `OnlineTicketBookingSystem_component.excalidraw`

- **Client → Virtual Waiting Room**: on sale opening, every arriving user joins a queue rather than hitting the booking flow directly (§5).
- **Waiting Room → Client**: admits users at a controlled rate, issuing a short-lived admission token.
- **Client (admitted) → Seat Map Service**: attempts to reserve a specific seat using the standard atomic conditional decrement.
- **Seat Map Service → Checkout**: on successful reservation, the usual payment flow (`PaymentProcessingSystem.md`) completes the purchase before the hold expires.

## 5. Deep Dive — Virtual Waiting Room / Admission Control — `OnlineTicketBookingSystem_sequence.excalidraw`

**Options considered:**
- Let all 500K users hit the real seat-reservation and checkout path simultaneously — correctness can still hold if the reservation logic itself is atomic (per `ECommercePlatform.md` §5), but the system burns its entire capacity on 500K simultaneous attempts, 90% of which are destined to fail, producing a slow, unfair experience where success depends on network latency rather than fairness.
- **Virtual waiting room admitting users at a controlled rate — the pick:** excess demand is held outside the booking system entirely; users are let into the real seat-selection flow only as fast as it can actually process them, turning an uncontrolled thundering herd into an orderly, bounded-concurrency flow. The seat reservation step itself is unchanged — still the same atomic conditional decrement, just per-seat instead of per-SKU-quantity.

**Terms:**

| Term | Meaning |
|---|---|
| Virtual waiting room | Holds excess demand outside the real booking system, admitting users at a controlled, sustainable rate. |
| Admission token | A short-lived, single-use credential proving a user has been let into the booking flow — required to attempt a reservation. |
| Fair ordering | Admission order is queue-entry order (FIFO or randomized at entry), not first-to-hit-the-server — which otherwise favors whoever has the fastest script/network. |
| Seat-level atomic reservation | The same conditional-decrement mechanism as `ECommercePlatform.md` §5, applied to one specific seat rather than an SKU quantity count. |

**Worked timeline** (sale opens, 500K users arrive within 60s, 50,000 seats):

| Time | Event | Outcome |
|---|---|---|
| t=0 | 500K users join the waiting room simultaneously | none yet admitted; all queued fairly |
| t=0–25s | Waiting room admits users at a sustainable rate (e.g. 2,000/sec), each gets a one-time admission token | admitted users attempt real seat reservations |
| t≈25s | All 50,000 seats reserved and sold | remaining queued users told "sold out" immediately — no wasted reservation-system load on doomed attempts |

**Correctness pitfall:** admission tokens must be single-use and time-boxed — a token that can be reused or shared lets one user (or a script replaying it) make multiple reservation attempts while others who legitimately waited in line never get a turn, defeating the entire fairness guarantee the waiting room exists to provide.

## 6. Data Model

- Queue entry: `queue_token, event_id, joined_at, position`.
- Admission token: `token, user_id, issued_at, expires_at, used: bool` — checked and marked used atomically on first reservation attempt.
- Seat: `seat_id, event_id, status (available|reserved|sold), reservation_id, hold_expires_at` — same shape as `ECommercePlatform.md` §6's inventory split, at per-seat granularity.

## 7. Scaling, Failure & Trade-offs

- The waiting room's admission rate is tuned to match the downstream reservation system's actual sustainable throughput — a deliberate backpressure valve, the same principle as `LoggingEventCollectionPlatform.md` §5's buffering, just applied to user admission instead of log lines.
- A reservation that expires unconfirmed (user abandoned checkout) releases the seat back to `available`, identical to `ECommercePlatform.md` §7's reservation TTL handling.
- For extremely popular events, the waiting room itself must scale horizontally and be stateless beyond the shared queue-position store — admitting the wrong number of users (double-admitting past capacity) just shifts contention back to the seat-reservation layer, which is designed to handle it correctly either way.

## 8. Wrap-up

Delivered: a waiting-room admission layer that absorbs a flash-sale thundering herd fairly and without wasting capacity, sitting in front of the already-correct atomic seat-reservation mechanism. Not covered due to time: event/venue search, dynamic pricing, and refunds/resale.