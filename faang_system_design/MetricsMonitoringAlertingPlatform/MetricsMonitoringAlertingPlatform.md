# Metrics, Monitoring and Alerting Platform — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `MetricsMonitoringAlertingPlatform_component.excalidraw`, `MetricsMonitoringAlertingPlatform_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform that collects time-series metrics from distributed infrastructure and evaluates alert rules against them, protecting itself from the single most common real-world failure mode: a careless metric label blowing up storage for everyone.

**In scope:** metric ingestion, cardinality control, alert evaluation.

**Explicitly deferred:** dashboard/visualization UI, the alert-rule query language itself, notification delivery (own exercise — see `NotificationService`-style patterns in `NewsFeedSystem.md`'s fan-out design for the general shape).

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~20M active time series, 10-second scrape interval (~2M samples/sec ingested).
- **Retention:** 13 months, via downsampling of older data.
- **Latency:** alert-evaluation latency (breach → notification) p99 < 30 s.
- **Availability:** the platform must stay available *during* the infrastructure incidents it exists to detect.

**Capacity check:** 2M samples/sec is a routine time-series ingestion rate — the actual risk to this NFR isn't steady-state volume, it's a single bad metric label choice multiplying the *number of distinct series* by orders of magnitude overnight, which no amount of steady-state capacity planning protects against on its own.

## 3. Interface

```
POST /metrics {name, labels, value, timestamp} -> ack
GET /query?promql_like_expr=<...>&time_range=<...> -> series[]
```

## 4. High-Level Architecture — `MetricsMonitoringAlertingPlatform_component.excalidraw`

- **Scraper/Agent → Ingestion Layer**: pushes or is scraped for samples every 10s.
- **Ingestion Layer → Cardinality Guard**: validates each new series against its metric's label schema and cardinality limit *before* it's accepted (§5).
- **Ingestion Layer → Time-Series Store**: accepted samples land here, downsampled over time per the retention policy.
- **Alert Evaluator → Time-Series Store**: periodically evaluates alert rules against recent data.
- **Alert Evaluator → Notification Dispatcher**: fires on a breach.

## 5. Deep Dive — Cardinality Explosion Control — `MetricsMonitoringAlertingPlatform_sequence.excalidraw`

**Options considered:**
- No limits on label cardinality — simplest for engineers emitting metrics, but a single careless label (e.g. tagging a metric with a raw `user_id` or request ID) multiplies the number of distinct time series combinatorially — potentially by orders of magnitude for one metric name — overwhelming storage and starving every other legitimate metric on the same cluster.
- **Cardinality limits + label schema enforced at ingestion — the pick:** the platform tracks the number of distinct label-value combinations per metric name and rejects new series beyond a configured ceiling; label keys are validated against a pre-registered schema before being accepted at all.

**Terms:**

| Term | Meaning |
|---|---|
| Time series | A unique combination of metric name + all label key/value pairs. |
| Cardinality | The total number of distinct time series a metric name expands into — the product of each label's distinct value count. |
| Cardinality limit | A per-metric ceiling on distinct series, enforced at ingestion. |
| Label schema | Pre-registered label keys for a metric; unregistered or unbounded-value keys (like raw IDs) are rejected. |

**Worked timeline** (`http_requests_total{method, status}`, cardinality limit 10,000):

| Time | Event | Cardinality | Outcome |
|---|---|---|---|
| t=0 | Normal: 5 methods × 5 statuses | 25 series | healthy |
| t=1 | A bad deploy adds a `user_id` label by mistake | would expand toward tens of millions | Ingestion checks against the limit |
| t=1 | Limit (10,000) exceeded | new label combinations rejected | storage protected; existing 25 series keep working normally |

**Correctness pitfall:** enforcing the cardinality limit by silently dropping samples once the ceiling is hit, without surfacing that fact as its own metric, hides the problem from exactly the team whose dashboards are now silently going blank or wrong. The drop itself must be observable — e.g. a `dropped_samples_total` counter emitted by the ingestion layer itself — otherwise someone ships a bad label and nobody finds out why their data looks broken.

## 6. Data Model

- Time-series store: `(metric_name, label_set) -> [(timestamp, value), ...]`, downsampled progressively as data ages (e.g. raw for 24h, 5-minute rollups beyond that).
- Label schema registry: `metric_name -> {allowed_label_keys, cardinality_limit}`.
- Dropped-sample counter: `metric_name -> dropped_count` — itself a first-class metric, not a side log.

## 7. Scaling, Failure & Trade-offs

- The platform's own metrics (about itself) are stored and alertable through the same pipeline, with enough isolation (separate storage/compute tier) that a cardinality incident on *user* metrics doesn't also take down the platform's self-monitoring.
- Alert evaluation is read-heavy and can lag slightly behind real-time without violating the 30s target, since it only needs "recent enough" data, not the absolute latest sample.
- Downsampling trades precision for storage cost over the 13-month retention window — recent data stays at full resolution for debugging, older data is aggregated since nobody needs second-level precision on metrics from 11 months ago.

## 8. Wrap-up

Delivered: an ingestion-time cardinality guard that prevents a single bad label from taking down shared infrastructure, with an explicit requirement that any enforced drop be observable rather than silent. Not covered due to time: the dashboard UI, the alert-rule query language, and notification delivery mechanics.