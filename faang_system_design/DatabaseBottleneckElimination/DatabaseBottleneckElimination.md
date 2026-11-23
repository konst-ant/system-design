# Eliminate a Database Bottleneck in an Existing System — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `DatabaseBottleneckElimination_component.excalidraw`, `DatabaseBottleneckElimination_sequence.excalidraw`. Part II — Architectural Evolution: database utilization has become the primary bottleneck as traffic grows.*

## 1. Problem & Scope

Several application instances share a central relational database; database utilization has become the primary bottleneck. Diagnose the actual saturated resource before applying a fix, and restore headroom without a full rewrite or extended downtime.

**In scope:** diagnostic methodology, matching the fix to the diagnosed cause, validating the fix with metrics.

**Explicitly deferred:** a full re-architecture to a different database engine, data-layer rewrites, long-term capacity planning beyond the stated 3x headroom target.

## 2. Current State & Targets

- **Current:** traffic has grown to ~50K QPS against the primary database, running at 90%+ CPU; p99 query latency has degraded from 20 ms to 800 ms over two quarters; periodic connection-pool exhaustion at peak.
- **Target:** restore p99 < 50 ms with headroom to 3x current peak, without a full rewrite or extended downtime.

**Capacity check:** three symptoms are present at once (high CPU, high latency, connection exhaustion) — the instinct to treat these as three separate problems needing three separate fixes is exactly the trap this exercise is designed to test; they're plausibly all downstream of one root cause.

## 3. Interface

```
(Unchanged client-facing API — the fix is intended to be transparent to callers.)
Internal diagnostic: EXPLAIN ANALYZE <query>, connection pool metrics, slow-query log aggregated by total time
```

## 4. High-Level Architecture — `DatabaseBottleneckElimination_component.excalidraw`

- **App Instances → Connection Pool → Primary Database**: the current, unchanged topology.
- **Query Profiler** (diagnostic, not a new runtime component): aggregates the slow-query log by total aggregate time per query pattern, not just by slowest individual query (§5).
- **Primary Database → New Index**: the targeted fix, applied once the root cause is confirmed.
- **(Only if diagnosis demands it) Read Replica**: offloads read traffic — added *after* confirming reads, not writes, are the saturated resource.

## 5. Deep Dive — Diagnose Before You Fix — `DatabaseBottleneckElimination_sequence.excalidraw`

**Options considered:**
- Jump straight to the most commonly-reached-for fix (add a read replica, or bump the connection pool size) without first confirming which resource is actually saturated — risky, because the wrong fix either doesn't help (a replica doesn't help if the real problem is write-side lock contention) or actively makes things worse (a bigger pool against an already-CPU-saturated database just lets more concurrent slow queries pile onto the same saturated CPU).
- **Pick: profile first to identify the specific saturated resource, then apply the matching fix.** Aggregate the slow-query log by *total* time (frequency × per-query cost), not by the single slowest query, since a cheap query run a million times can cost more aggregate CPU than an expensive query run once.

**Terms:**

| Term | Meaning |
|---|---|
| Query profiling | Identifying which query *pattern* consumes the most aggregate DB time — not the slowest single instance. |
| Connection pool exhaustion | Requests queueing for a DB connection itself — a distinct symptom from slow query execution, often a downstream effect of it. |
| Missing index | A query doing a full table scan where an index would let it seek directly — often the single highest-leverage fix. |
| Workload separation | Routing analytical/reporting queries to a separate replica only once shown to interfere with user-facing transactional queries. |

**Worked timeline:**

| Step | Action | Result |
|---|---|---|
| 1 | Profile queries by aggregate time | One pattern — `SELECT * FROM posts WHERE user_id = ? ORDER BY created_at`, no index on `(user_id, created_at)` — accounts for 60% of total DB time via full table scans |
| 2 | Add the missing composite index | That query's cost drops from O(table size) to O(matching rows) |
| 3 | Re-measure | CPU drops from 90%+ to ~40%; p99 drops from 800 ms to ~60 ms |
| 4 | Re-check connection pool | Exhaustion also resolves — it was a **symptom** of slow queries holding connections longer, not an independent problem |

**Correctness pitfall:** increasing the connection pool size or adding a read replica as the *first* response — without profiling to confirm those are the actual bottleneck — can make things worse, not better. A bigger pool against an already-CPU-saturated database increases contention on that same saturated resource rather than relieving it. The fix must match the diagnosed resource, not the most commonly-reached-for remedy.

## 6. Data Model

- Slow-query aggregate: `query_pattern -> {call_count, total_time, avg_time}` — the diagnostic artifact that drives the fix, not a new runtime table.
- New index: `(user_id, created_at)` on the `posts` table — the actual schema change, minimal and targeted.
- Connection pool metrics (existing instrumentation, not new): `active_connections, queued_requests, wait_time`.

## 7. Scaling, Failure & Trade-offs

- Adding an index is not free — it costs write-path overhead (every insert/update now also maintains the index) and storage; validated against the 3x-headroom target to confirm the trade is worth it, which it is here since reads vastly outnumber writes on this table.
- The fix is applied without extended downtime because adding an index concurrently (most databases support building an index without locking the table for writes) doesn't require a maintenance window — a materially different operation from a schema migration (see `ZeroDowntimeDatabaseMigration.md` for that harder case).
- If profiling had instead shown the bottleneck was genuinely write-side (e.g. excessive write volume, not query inefficiency), the correct next step would be partitioning or workload separation — not an index, which only helps reads. The point of diagnosing first is exactly to avoid applying the wrong one of these.

## 8. Wrap-up

Delivered: a diagnose-before-fix methodology that correctly identified a single root cause (a missing index) behind three symptoms that looked like independent problems, validated by before/after metrics. Not covered due to time: database engine migration, long-term capacity planning, and deeper workload-separation architecture.