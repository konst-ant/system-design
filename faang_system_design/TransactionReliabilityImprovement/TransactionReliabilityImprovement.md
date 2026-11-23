# Improve Transaction Reliability Across Services — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `TransactionReliabilityImprovement_component.excalidraw`, `TransactionReliabilityImprovement_sequence.excalidraw`. Part II — Architectural Evolution: a distributed transaction spans multiple services and occasionally leaves them inconsistent.*

## 1. Problem & Scope

A business operation (e.g. "place order") touches its own database *and* publishes an event (or calls another service) to trigger downstream work. Under failure (crash between the DB write and the publish, or a publish that silently fails) services drift out of sync. Eliminate that drift without introducing a slow, lock-heavy distributed transaction.

**In scope:** the dual-write problem, a pattern that makes the DB write and the event publish atomic in effect, idempotent consumption downstream.

**Explicitly deferred:** full two-phase-commit/XA transactions (rejected up front — see §5), cross-service choreography of a multi-step business process (see `OnlineTicketBookingSystem.md`/`HotelFlightReservationSystem.md` for saga-based orchestration of that harder problem).

## 2. Current State & Targets

- **Current:** 5 services, ~5K TPS combined, each service writes to its own database then separately publishes an event to notify downstream services; roughly 0.1% of transactions end up inconsistent (DB committed, event never published, or vice versa) — usually surfacing as "order placed but no confirmation email," or "inventory decremented but no order record."
- **Target:** drive inconsistency to effectively zero (bounded only by downstream processing bugs, not by the publish mechanism itself), with publish latency still in the tens-of-milliseconds range — no distributed-lock slowdown.

**Capacity check:** at 5K TPS, 0.1% inconsistent is ~5 broken transactions per second — small as a percentage, large as an absolute, continuous volume of support tickets and manual reconciliation; this is why it reads as "occasional" but is in practice an operational fire that never goes out.

## 3. Interface

```
POST /orders -> {order_id, status}   (unchanged client-facing API)
Internal: outbox table row -> relay -> message broker topic "order.events"
Consumer contract: process(event) must be idempotent, keyed by event_id
```

## 4. High-Level Architecture — `TransactionReliabilityImprovement_component.excalidraw`

- **Service → Local Database**: writes business state and an outbox row in the **same local transaction** (§5).
- **Outbox Table → Relay**: a separate process polls (or tails the DB's change log) and publishes rows to the broker.
- **Relay → Message Broker**: publishes at-least-once; the relay retries until the broker acknowledges, then marks the row published.
- **Message Broker → Downstream Consumers**: each consumer processes idempotently, keyed by `event_id` (§5).

## 5. Deep Dive — Transactional Outbox — `TransactionReliabilityImprovement_sequence.excalidraw`

**Options considered:**
- Two-phase commit (XA transaction) across the database and the message broker: makes the write and the publish genuinely atomic, but requires the broker to participate in a distributed transaction protocol — most brokers don't support this, it holds locks for the duration of the slowest participant, and it doesn't scale to 5K TPS with acceptable latency.
- Write to the DB, then publish, with no further protection ("dual write"): simplest to write, but a crash between the two steps (or a publish that times out ambiguously) leaves the two systems inconsistent — this is the current state and the actual bug being fixed.
- **Pick: Transactional Outbox.** Write the business row *and* an outbox row describing the event in the same local ACID transaction — so either both happen or neither does, using a guarantee the database already provides for free. A separate relay process then reads unpublished outbox rows and publishes them, retrying until acknowledged, and only then marks them published.

**Terms:**

| Term | Meaning |
|---|---|
| Outbox table | A table in the same database as the business data; each row represents one event still needing to be published. |
| Relay | A separate process (polling the table, or tailing the DB's write-ahead log via CDC) that publishes outbox rows to the broker. |
| At-least-once delivery | The relay retries until acknowledged, so a crash mid-publish can cause a duplicate delivery — never a lost one. |
| Idempotent consumer | A downstream consumer that de-duplicates by `event_id`, so processing the same event twice has the same effect as processing it once. |
| Dual-write problem | Writing to two systems (DB, broker) as two separate operations with no shared transaction — the root problem the outbox pattern eliminates. |

**Worked timeline:**

| Step | Action | Result |
|---|---|---|
| 1 | Service receives "place order"; writes `orders` row + `outbox` row in one local transaction | Both committed together or neither is — no partial state is possible |
| 2 | Service crashes immediately after commit, before responding to the caller | On restart, the committed outbox row is still there — nothing was lost |
| 3 | Relay polls, finds the unpublished outbox row, publishes to the broker | Broker acknowledges |
| 4 | Relay marks the outbox row published | Row excluded from future polls |
| 5 | Relay crashes *after* publishing but *before* marking the row published | On restart, relay re-publishes the same event — broker delivers it twice |
| 6 | Downstream consumer checks `event_id` against its own dedup store, sees it already processed | Second delivery is a no-op — no double-charge, no duplicate email |

**Correctness pitfall:** the outbox pattern guarantees at-least-once delivery, not exactly-once — step 5/6 above is not a rare edge case, it's an expected, routine occurrence under this design. Skipping idempotent consumption because "the outbox already solved reliability" reintroduces the exact inconsistency this pattern exists to remove, just one hop downstream. This is the same idempotency-key discipline used in `PaymentProcessingSystem.md` §5: the hard guarantee is pushed to the one layer (the consumer) that can actually enforce exactly-once *effect* even though delivery itself is only at-least-once.

## 6. Data Model

```
outbox_events: {
  event_id (uuid, pk), aggregate_type, aggregate_id,
  payload (jsonb), created_at, published_at (nullable)
}
-- written in the SAME transaction as the business row, e.g.:
-- BEGIN; INSERT INTO orders (...); INSERT INTO outbox_events (...); COMMIT;
```
- Consumer-side dedup store: `event_id -> processed_at`, checked before applying any event's effect — the mechanism that makes at-least-once delivery behave like exactly-once processing.

## 7. Scaling, Failure & Trade-offs

- Polling the outbox table adds load proportional to poll frequency; at higher scale, tailing the database's change log (CDC, e.g. Debezium-style) avoids polling entirely and reduces publish latency from "next poll interval" to near-real-time — a trade-off between operational simplicity (polling) and latency/load (CDC), made based on actual publish-latency requirements.
- The outbox table grows unboundedly without cleanup; published rows are deleted or archived on a delay (keeping a short recent window for debugging) rather than immediately, to avoid racing the relay.
- This pattern solves the dual-write problem for a *single* service's local transaction plus publish. It does not, by itself, coordinate a multi-step business process spanning several services — that's the saga pattern (`OnlineTicketBookingSystem.md`/`HotelFlightReservationSystem.md`), which can itself be built using outbox-published events as its reliable signaling mechanism underneath.

## 8. Wrap-up

Delivered: a transactional outbox design that makes the database write and the event publish atomic in effect (by making them one local transaction) and pushes the remaining at-least-once-delivery risk to idempotent, dedup-keyed consumers. Not covered due to time: multi-service saga orchestration, CDC-based relay implementation details, and outbox table retention policy specifics.