# Logging and Event Collection Platform — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `LoggingEventCollectionPlatform_component.excalidraw`, `LoggingEventCollectionPlatform_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform that collects logs from thousands of applications and infrastructure components, guaranteeing no log loss during a downstream outage — including during the incident that's generating the burst of logs in the first place.

**In scope:** agent-side buffering and backpressure, ingestion, retention tiers.

**Explicitly deferred:** query/search engine internals (own exercise — see `SearchEngine.md` for the general indexing approach), structured log parsing rules, alerting on log content (see `MetricsMonitoringAlertingPlatform.md` for the alerting model itself).

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~50,000 hosts/services emitting ~10 TB/day raw volume, bursting to 3x during incidents.
- **Latency:** ingestion-to-queryable p99 < 30 s.
- **Retention:** 30 days hot, 1 year cold/archive.
- **Durability:** no log loss during a downstream outage of several minutes — logs must be buffered, not dropped.

**Capacity check:** the exact moment log volume triples (an incident) is also, ironically, when the downstream collector is most likely to be stressed — this isn't two independent risks, it's one correlated failure mode, which is precisely why "buffer for several minutes of 3x burst" has to be sized deliberately, not assumed away as a rare edge case.

## 3. Interface

```
Agent -> Collector: batched log lines (async, buffered)
GET /search?query=<...>&time_range=<...> -> matching log entries
```

## 4. High-Level Architecture — `LoggingEventCollectionPlatform_component.excalidraw`

- **Application → Log Agent**: writes log lines locally; the application never blocks on logging.
- **Log Agent → Local Disk Buffer**: durable queue, survives agent restart.
- **Log Agent → Collector**: ships batched lines asynchronously; retries with backoff if the collector is unavailable.
- **Collector → Hot Index**: recent logs (30 days), queryable.
- **Archiver → Cold Storage**: older logs migrate out of the hot index to cheaper storage (1 year retention).

## 5. Deep Dive — Agent-Side Buffering & Backpressure — `LoggingEventCollectionPlatform_sequence.excalidraw`

**Options considered:**
- Synchronous send, one request per log line, no local buffer — simplest, but if the collector is briefly slow or down (more likely exactly during an incident), every logging call either blocks the application or silently drops the line — neither is acceptable.
- **Local durable (disk-backed) buffer + batched async shipping — the pick:** the application writes to a local buffer and returns immediately; the agent batches and ships asynchronously, and if the collector is unavailable, the buffer absorbs the backlog up to a disk-space limit sized for *burst*, not average, volume.

**Terms:**

| Term | Meaning |
|---|---|
| Log agent | Runs alongside the application; collects, buffers, and ships logs. |
| Disk-backed buffer | Durable local queue — survives an agent restart, unlike an in-memory-only buffer. |
| Backpressure | The agent slows/stops accepting new logs locally only once its own buffer nears full — it never blocks the application's hot path. |
| Batch shipping | Grouping many log lines into one network call for throughput. |

**Worked timeline** (buffer sized for 10 minutes of 3x burst volume):

| Time | Event | Buffer state |
|---|---|---|
| t=0 | Normal operation: ship every 1s, buffer stays near-empty | healthy |
| t=1 | Incident hits: log volume triples *and* the collector slows down | buffer starts growing (produce rate > ship rate) |
| t=1–10min | Buffer absorbs the surge using its sized disk allocation | growing but within budget |
| t=10min | Collector recovers; agent drains the backlog | draining, catches up without ever blocking the app or dropping a line |

**Correctness pitfall:** sizing the buffer for *average* volume (not burst) and/or keeping it in-memory only means the worst-case scenario — a major incident spiking volume while also stressing the collector — is precisely when the buffer overflows, or an agent/application restart during the incident wipes an in-memory-only buffer, losing the exact diagnostic logs needed to understand the incident. The buffer must be sized for burst volume and durable across restarts; that's why "no log loss during a downstream outage of several minutes" is stated as a hard NFR, not an aspiration.

## 6. Data Model

- Hot index: `log_id, host, service, timestamp, structured_fields, raw_text` — indexed for the 30-day retention window.
- Cold archive: same records, compressed, in cheap object storage, retrievable but not directly indexed for fast search.
- Agent buffer: on-disk append log of unshipped lines, with a configured max size (sized for burst, per §5).

## 7. Scaling, Failure & Trade-offs

- Collectors scale horizontally behind a load balancer; agents retry with backoff across available collector instances, so one collector instance failing doesn't stall agents pointed at it.
- The 30-day hot / 1-year cold split trades query latency (cold storage is slower/batch-oriented) for cost — most real queries target recent data anyway, matching the Zipfian-style recency skew seen in `SearchEngine.md` §7's query cache reasoning.
- An agent that's buffering during a prolonged outage eventually hits its disk limit; at that point the policy choice is drop-oldest vs. drop-newest vs. stop-accepting — an explicit, product-level decision, not a default to leave unexamined.

## 8. Wrap-up

Delivered: a durable, burst-sized agent buffer that decouples application logging from collector availability, with an explicit sizing rationale tied to the exact failure mode that matters most (incidents). Not covered due to time: search/query internals, structured parsing rules, and content-based alerting.
