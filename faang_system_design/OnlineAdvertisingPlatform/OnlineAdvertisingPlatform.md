# Online Advertising Platform — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `OnlineAdvertisingPlatform_component.excalidraw`, `OnlineAdvertisingPlatform_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform that selects an ad for each incoming placement request under an extremely tight latency budget, while keeping every advertiser's spend within a few percent of their configured daily budget despite massive concurrent auction volume.

**In scope:** the auction path, budget pacing under concurrency.

**Explicitly deferred:** targeting/relevance model internals, fraud detection, billing reconciliation batch jobs.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~1M ad-auction requests/sec at peak.
- **Latency:** each auction (targeting → bidding → selection) must complete within a ~100 ms SLA.
- **Correctness:** budget pacing must track the configured daily budget within a few percent; impression/click events for billing must be processed at comparable throughput with no double-counting.

**Capacity check:** this NFR explicitly tolerates "a few percent" of budget error — that single sentence is the whole design driver. It means approximate, high-throughput concurrency control is not a compromise here, it's the *correct* engineering choice, unlike `ECommercePlatform.md` or `DigitalWalletService.md` where the correctness bar (zero overselling, zero negative balance) leaves no such room.

## 3. Interface

```
POST /auction {placement_context} -> {winning_ad_id, price}  // must return within 100ms
POST /events {ad_id, type: impression|click} -> ack
```

## 4. High-Level Architecture — `OnlineAdvertisingPlatform_component.excalidraw`

- **Ad Request → Serving Node**: one of many stateless nodes running the auction end-to-end within the 100ms budget.
- **Serving Node → Targeting/Bidding**: selects eligible ads and computes bids — internals deferred.
- **Serving Node → Budget Lease Manager**: checks out a small local slice of the winning advertiser's remaining budget (§5), spent without a network call per auction.
- **Budget Lease Manager → Authoritative Budget Store**: periodically reconciles each node's local spend.

## 5. Deep Dive — Budget Lease & Pacing Under Concurrency — `OnlineAdvertisingPlatform_sequence.excalidraw`

**Options considered:**
- Atomic conditional decrement against the exact remaining budget for every single winning bid — perfectly accurate (same mechanism as `ECommercePlatform.md`/`DigitalWalletService.md`), but at 1M auctions/sec, funneling every win for a popular advertiser through one atomically-guarded counter creates severe hot-key contention — exactly the bottleneck those other systems accept because their correctness bar leaves no alternative. Here, it doesn't.
- **Locally-cached budget lease per serving node, spent down locally, synced periodically — the pick:** each node checks out a small slice of an advertiser's remaining daily budget and spends it without a network round-trip per auction — the same lease pattern as `RateLimiter.md` §5's local token-bucket, now applied to money, and deliberately accepting the same bounded overshoot this NFR explicitly allows.

**Terms:**

| Term | Meaning |
|---|---|
| Budget lease | A chunk of an advertiser's remaining daily budget checked out by one serving node to spend locally. |
| Pacing | Deliberately spreading an advertiser's budget across the day rather than spending it all immediately — related to, but distinct from, pure concurrency control. |
| Overshoot bound | The "within a few percent" tolerance that justifies the lease design — mirrors `RateLimiter.md` §5's accepted bounded overshoot exactly. |
| Sync interval | How often a node reconciles its local spend against the authoritative store — the same latency/accuracy tuning knob as `RateLimiter.md` §5. |

**Worked timeline** (advertiser's daily budget = $10,000):

| Time | Event | Outcome |
|---|---|---|
| t=0 | Serving Node A checks out a $50 lease | spends it down across several wins, zero network calls per auction |
| t=0 | Serving Node B, concurrently, checks out its own $50 lease | same — independent, no coordination with A |
| every few sec | Each node syncs its local spend back to the authoritative store | drift stays bounded by lease size × active node count |

**Correctness pitfall:** sizing lease amounts without regard to the number of concurrently-active serving nodes (a fixed $50 lease regardless of whether 10 or 10,000 nodes are serving that advertiser) makes the overshoot bound scale with fleet size, not with a deliberately chosen tolerance. Lease size must be derived from the advertiser's remaining budget divided by a function of active node count and sync interval — exactly mirroring how `RateLimiter.md` §5 derived its lease size from `rate × sync_interval`, never picked arbitrarily.

## 6. Data Model

- Authoritative budget: `advertiser_id -> {daily_budget, spent_today}`, updated only by lease reconciliation, never by individual auctions directly.
- Local lease: `serving_node_id, advertiser_id, lease_amount, remaining`.
- Event log: `ad_id, type, timestamp, dedup_key` — impressions/clicks processed with the same idempotency-key discipline as `PaymentProcessingSystem.md` §5, to avoid double-billing an advertiser for one impression counted twice.

## 7. Scaling, Failure & Trade-offs

- A serving node crashing with an unspent partial lease simply loses that small remainder — bounded waste, the same "discard on expiry" trade-off as `RateLimiter.md` §5, now in dollars rather than requests.
- Near end-of-day, lease sizes can shrink (smaller slices, more frequent syncs) to tighten the overshoot bound exactly when "ran over budget" would otherwise be most visible to the advertiser.
- Impression/click billing events, unlike the auction path itself, can tolerate a slightly longer processing latency — they flow through a durable, idempotent-keyed pipeline rather than needing the 100ms synchronous guarantee the auction does.

## 8. Wrap-up

Delivered: a lease-based budget pacing design that trades strict per-auction exactness for the throughput the 100ms/1M-QPS auction path requires, justified directly by the NFR's own tolerance for a bounded overshoot. Not covered due to time: targeting/relevance internals, fraud detection, and billing reconciliation.