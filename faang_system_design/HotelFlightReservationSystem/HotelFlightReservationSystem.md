# Hotel and Flight Reservation System — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `HotelFlightReservationSystem_component.excalidraw`, `HotelFlightReservationSystem_sequence.excalidraw`.*

## 1. Problem & Scope

Design a booking platform that searches and books inventory from many independent external suppliers, guaranteeing a booking never leaves a held-but-unpaid reservation or a charge with no corresponding booking when one step of a multi-step booking fails.

**In scope:** multi-supplier search fan-out, the compensating-transaction pattern for bookings that span independent external systems.

**Explicitly deferred:** pricing/rate-shopping logic, loyalty programs, the payment provider integration itself (own exercise — see `PaymentProcessingSystem.md`, which this system calls into as one saga step).

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~10M searches/day, fanning out to hundreds of external supplier APIs per search.
- **Latency:** search p99 < 2 s despite the fan-out; booking-confirmation p99 < 5 s.
- **Correctness:** no double-booking, no payment taken without a corresponding confirmed reservation — even when a multi-step booking partially fails across independent suppliers.

**Capacity check:** a booking here is never one atomic operation — it's a sequence of calls to independent external systems (a hotel supplier, a payment provider) with no shared transaction coordinator between them. The real engineering problem is what happens when step 2 fails *after* step 1 already succeeded at an external supplier — there's no database ROLLBACK that can undo an already-confirmed external hotel reservation.

## 3. Interface

```
GET /search?destination=<...>&dates=<...> -> {results: [{supplier, offer_id, price}]}
POST /bookings {offer_id, payment_info} -> {booking_id, status}
```

## 4. High-Level Architecture — `HotelFlightReservationSystem_component.excalidraw`

- **Client → Search Aggregator**: fans out to hundreds of supplier APIs in parallel, merges results (same scatter-gather shape as `SearchEngine.md` §5b, bounded per-supplier timeout instead of per-shard).
- **Client → Booking Orchestrator**: drives the multi-step booking saga (§5).
- **Booking Orchestrator → Supplier API**: step 1 — reserve the room/seat.
- **Booking Orchestrator → Payment Service** (`PaymentProcessingSystem.md`): step 2 — charge the customer.
- **Booking Orchestrator → Compensation Log**: records which steps have completed, so a later failure knows exactly what to undo.

## 5. Deep Dive — Saga Pattern with Compensating Transactions — `HotelFlightReservationSystem_sequence.excalidraw`

**Options considered:**
- Treat the whole booking (reserve + charge) as if it were one atomic transaction — doesn't work, because the hotel supplier and the payment provider are separate external systems with no shared transaction coordinator; there's no way to "rollback" an already-confirmed external reservation via a database transaction.
- **Saga — the pick:** each step is a local, independently-committed action, paired with a defined compensating action that undoes it if a *later* step in the same booking fails. If step 2 fails, the orchestrator runs step 1's compensation automatically.

**Terms:**

| Term | Meaning |
|---|---|
| Saga | A sequence of local transactions, each with a corresponding compensating action, used when no single ACID transaction can span all the systems involved. |
| Compensating transaction | The specific action that semantically undoes an already-committed step — not a generic rollback, since the step already happened in a real external system. |
| Saga orchestrator | Tracks which steps of a given booking have completed and which compensations to run if a later step fails. |
| Forward vs. backward recovery | Retrying a failed step forward (if safely retryable) vs. running compensations backward — the choice depends on the failed step itself. |

**Worked timeline** (Booking B: reserve room at Supplier X, then charge payment):

| Time | Event | Outcome |
|---|---|---|
| t=0 | Step 1: reserve room at Supplier X | succeeds — room held in Supplier X's system |
| t=1 | Step 2: charge customer's payment | **fails** (card declined) |
| t=1 | Orchestrator detects step 2's failure | since step 1 already succeeded externally, runs step 1's compensation |
| t=1 | Compensation: cancel/release the room reservation at Supplier X | room becomes available again; customer told booking failed; no charge, no leftover hold |

**Correctness pitfall:** a compensating action can itself fail (e.g. the supplier's cancel-reservation API is briefly down). The orchestrator must retry the compensation until it succeeds, or escalate to manual intervention — never give up silently, because an un-compensated step leaves a real-world side effect (a held room) with no corresponding successful booking, exactly the inconsistent state a saga exists to prevent.

## 6. Data Model

- Saga instance: `booking_id, steps: [{step_name, status, compensation_status}], current_step`.
- Supplier reservation: `supplier, offer_id, external_reservation_ref, status`.
- Compensation log: append-only record of every compensation attempted and its outcome — the audit trail for "what did we undo, and did it actually succeed."

## 7. Scaling, Failure & Trade-offs

- Search fan-out tolerates individual slow suppliers the same way `SearchEngine.md` §5b tolerates slow shards — a bounded per-supplier timeout and a best-effort merged result, rather than blocking the whole search on one slow supplier.
- The saga orchestrator persists its state after every step (same durability principle as `WorkflowOrchestrationEngine.md` §5) so an orchestrator crash mid-booking doesn't lose track of which compensations are owed — it resumes from the last recorded step on restart.
- Suppliers with flaky or slow cancel-reservation APIs are a real operational risk the design must plan for explicitly (retry with backoff, alert if compensation stays unresolved past a threshold) rather than assume compensations always succeed promptly.

## 8. Wrap-up

Delivered: a saga-based booking flow with explicit compensating transactions that keeps a multi-supplier booking consistent even when a later step fails. Not covered due to time: pricing/rate-shopping, loyalty programs, and payment provider integration details (`PaymentProcessingSystem.md`).