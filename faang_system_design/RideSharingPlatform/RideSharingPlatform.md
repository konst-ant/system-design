# Ride-Sharing Platform — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `RideSharingPlatform_component.excalidraw`, `RideSharingPlatform_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform that matches riders with nearby drivers in real time, against a continuously-moving population of millions of drivers.

**In scope:** geospatial indexing of driver locations, nearby-driver query, match dispatch.

**Explicitly deferred:** fare calculation/surge pricing, trip-lifecycle/payment settlement (own exercise — see `PaymentProcessingSystem.md`), driver onboarding.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~5M active drivers pinging location every ~4s (~1.25M location writes/sec globally); ~50M riders.
- **Demand:** peak new-ride-request rate ~5K/sec in major metros.
- **Latency:** nearby-driver geo-query < 100 ms; match-to-driver p99 < 3 s.

**Capacity check:** a naive "scan every active driver and compute distance" approach is O(active driver count) per single ride request — with millions of drivers, even filtered to one city, that's far too slow to run per-request at 5K requests/sec with a 100ms budget; the query needs to touch a small, bounded number of candidates, not the whole population.

## 3. Interface

```
POST /drivers/{id}/location {lat, lng} -> ack  // every ~4s
POST /rides/request {pickup_lat, pickup_lng} -> {driver_id, eta}
```

## 4. High-Level Architecture — `RideSharingPlatform_component.excalidraw`

- **Driver App → Location Service**: pings current location every ~4s.
- **Location Service → Spatial Index**: updates the driver's cell membership on every ping (§5).
- **Rider App → Matching Service**: requests a ride from a pickup location.
- **Matching Service → Spatial Index**: queries nearby available drivers.
- **Matching Service → Driver App**: dispatches the match to the chosen driver.

## 5. Deep Dive — Geospatial Indexing via Geohash Grid — `RideSharingPlatform_sequence.excalidraw`

**Options considered:**
- Scan all active drivers and compute exact distance to each, keep the nearest — correct, but O(active driver count) per request; doesn't scale to millions of drivers and 5K requests/sec under a 100ms budget.
- **Geohash-based spatial grid — the pick:** the world is divided into cells via geohashing (a string encoding of lat/long where geographically close points share a common prefix); drivers are indexed by their current cell, and a "nearby drivers" query only looks at a small, bounded number of cells around the rider — the matching cell plus its immediate neighbors — never the whole driver population.

**Terms:**

| Term | Meaning |
|---|---|
| Geohash | A string encoding of a location such that nearby points tend to share a common prefix; precision is controlled by string length. |
| Grid cell | All drivers sharing the same geohash prefix at a chosen precision — the unit of the spatial index. |
| Neighbor cells | The ~8 cells surrounding a given cell, checked in addition to it, to avoid missing a driver who sits just across a cell boundary. |
| Driver location update | A driver's current cell membership, refreshed on every ~4s ping — removed from its old cell if it moved. |

**Worked timeline:**

| Time | Event | Outcome |
|---|---|---|
| t=0 | Driver D pings location, now in cell C | D's index entry moves from its old cell (if different) to cell C |
| t=1 | Rider R requests a ride at location L (geohashes to cell C) | Matching Service queries cell C + its 8 neighbor cells (9 total) |
| t=1 | Query returns 12 nearby drivers out of millions globally | all found in well under 100ms — the query never touched drivers outside those 9 cells |

**Correctness pitfall:** querying only the exact matching cell (not its neighbors) misses drivers who are geographically very close but sit just across a grid-cell boundary line — e.g. a driver 10 meters away in the adjacent cell would be invisible to a same-cell-only query, producing a worse match than one farther away purely due to an arbitrary grid line. The neighbor-cell check is required for correctness, not an optional optimization.

## 6. Data Model

- Spatial index: `geohash_cell -> [driver_ids currently in this cell]`, updated on every location ping.
- Driver state: `driver_id, current_cell, last_ping_at, availability`.
- Ride request: `rider_id, pickup_location, matched_driver_id, status`.

## 7. Scaling, Failure & Trade-offs

- Cell size (geohash precision) trades query breadth against candidate count — too coarse and a query returns too many candidates to rank quickly; too fine and legitimate nearby matches require checking more neighbor cells. Precision is tuned to the city's driver density.
- A driver whose location updates stop arriving (app backgrounded, connectivity lost) ages out of the index after a short TTL, the same stale-shared-state concern as `WebCrawler.md` §5's domain timer and `RealTimeChatSystem.md` §5's presence registry — without it, riders would keep getting matched to drivers who are no longer actually there.
- Dense urban cells (far more drivers per cell than a rural cell) can be further sub-divided dynamically, so no single cell becomes a hotspot of excessive candidates to rank per query.

## 8. Wrap-up

Delivered: a geohash grid index that bounds every "nearby drivers" query to a small, fixed number of cells regardless of total driver count, with an explicit correctness requirement for checking neighbor cells. Not covered due to time: fare/surge pricing, trip lifecycle and payment settlement, and driver onboarding.