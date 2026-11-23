# Multi-Region Distributed Architecture — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `MultiRegionArchitecture_component.excalidraw`, `MultiRegionArchitecture_sequence.excalidraw`. Part II — Architectural Evolution: an existing single-region application must expand to multiple regions.*

## 1. Problem & Scope

An application currently runs in one region. Expand it to multiple regions for availability and latency, with explicit recovery objectives and defined behavior during a network partition between regions — without silently risking split-brain writes.

**In scope:** region topology (active-active vs. active-passive), failover mechanics, split-brain prevention.

**Explicitly deferred:** per-service data-residency/compliance routing, CDN/static-asset specifics (see `VideoStreamingPlatform.md`/`LiveVideoStreamingService.md` for CDN patterns), cost modeling of cross-region bandwidth.

## 2. Current State & Targets

- **Current:** single region; ~20M DAU; p99 latency 150 ms for in-region users, 600+ ms for users on other continents.
- **Target:** expand to 3 regions; p99 < 150 ms for 95% of global users; **RTO < 5 minutes**, **RPO < 1 minute** for regional failover; defined, safe behavior during an inter-region network partition.

**Capacity check:** the target RPO (< 1 minute) directly bounds how asynchronous cross-region replication is allowed to be — replication lag becomes the data-loss window during a failover, so the replication design and the RPO target are the same number viewed from two sides.

## 3. Interface

```
Client -> Global Traffic Router -> nearest healthy region's API (unchanged from single-region API)
Internal: Region Health Check -> {region_id, status, last_heartbeat}
```

## 4. High-Level Architecture — `MultiRegionArchitecture_component.excalidraw`

- **Client → Global Traffic Router** (geo-DNS/anycast): routes to the nearest healthy region.
- **Region (each) → Local API tier**: serves reads locally in any region; writes route to the current primary (§5).
- **Primary Region → Standby Regions**: asynchronous replication, bounding the RPO.
- **Quorum/Arbiter Service**: independent of any single region, used to confirm a region is genuinely down before failover (§5).

## 5. Deep Dive — Active-Passive Failover with Quorum Fencing — `MultiRegionArchitecture_sequence.excalidraw`

**Options considered:**
- Active-active: all 3 regions accept writes locally, replicated bidirectionally — lowest write latency everywhere, but during a network partition between regions, two regions can accept conflicting writes to the same record with no clean way to merge most transactional data afterward (unlike `CollaborativeDesignCanvas.md` §5's CRDT model, which only works because its data is specifically designed to merge).
- **Active-passive per partition, with quorum-confirmed failover — the pick:** one region is the designated primary for writes at a time; standbys replicate asynchronously. Promoting a standby requires a quorum/independent arbiter to confirm the old primary is genuinely unreachable — never a single standby's own unilateral judgment — which rules out two regions simultaneously believing they're primary.

**Terms:**

| Term | Meaning |
|---|---|
| Primary region | The current authoritative region for writes on a given data partition. |
| Standby region | Replicates from the primary asynchronously; a candidate for promotion. |
| Failover | Promoting a standby to primary — must follow a quorum confirmation, not immediate reaction to unreachability. |
| Fencing | Preventing an old primary that becomes reachable again after a failover from accepting writes and conflicting with the newly promoted primary. |
| RTO / RPO | Recovery Time Objective (max downtime) / Recovery Point Objective (max data loss, bounded by replication lag). |

**Worked timeline** (Region A primary, B/C standbys, ~500ms replication lag):

| Time | Event | Outcome |
|---|---|---|
| t=0 | Region A becomes unreachable (regional outage) | B and C detect missed heartbeats, but do **not** promote immediately |
| t=0–90s | Quorum check: a majority of regions/arbiter confirm A is genuinely down, not just partitioned from one observer | confirmation reached |
| t=90s | B promoted to primary; A is fenced (rejects writes if it ever comes back without being re-admitted) | well within the 5-minute RTO target |
| — | Writes to A not yet replicated to B before the outage (≤ ~500ms–1s of data) | lost — this is the accepted RPO |

**Correctness pitfall:** promoting a standby based solely on "I can't reach the primary" — without a quorum or independent arbiter — is unsafe. If the real problem is a network partition isolating the *observer*, not the primary, the primary may still be alive and serving writes; now two regions believe they're primary simultaneously (split-brain), accepting conflicting writes with no reliable way to reconcile them later. The confirmation must come from a majority view or an independent arbiter, never a single standby's own judgment.

## 6. Data Model

- Region health: `region_id -> {status, last_heartbeat, is_primary_for: [partitions]}`.
- Replication state: `partition -> {primary_region, standby_regions, replication_lag}`.
- Fencing token: a monotonically increasing value issued on every promotion, attached to writes, so a stale "primary" presenting an old token is rejected outright by storage — the same fencing-token principle used for distributed locks.

## 7. Scaling, Failure & Trade-offs

- Reads are served from any region (eventually consistent) since they vastly outnumber writes in most workloads — the same read-heavy skew assumption used throughout this project (e.g. `SearchEngine.md`, `NewsFeedSystem.md`); only writes pay the cross-region latency cost of routing to the primary.
- A network partition that heals on its own (rather than being a genuine regional outage) must not trigger an unnecessary failover — the quorum check protects against both false positives (failing over when unneeded) and split-brain (failing over unsafely).
- Different data partitions can have different primary regions (e.g. EU users' primary in the EU region), so "failover" is a per-partition operation, not an all-or-nothing global event.

## 8. Wrap-up

Delivered: an active-passive, quorum-fenced failover design meeting explicit RTO/RPO targets while structurally ruling out split-brain writes during a network partition. Not covered due to time: data-residency routing, CDN specifics, and cross-region bandwidth cost modeling.
