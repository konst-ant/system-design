# Real-Time Data Ingestion Pipeline — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `RealTimeDataIngestionPipeline_component.excalidraw`, `RealTimeDataIngestionPipeline_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform that ingests high-volume event streams and makes them available for downstream processing, correctly handling events that arrive late or out of order relative to when they actually happened.

**In scope:** ingestion/buffering, event-time vs. processing-time handling, windowed aggregation correctness under late data.

**Explicitly deferred:** schema registry/evolution (own exercise — see `DistributedMessageQueue.md` for the underlying transport), the specific aggregation logic beyond windowing mechanics, exactly-once sink semantics.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~5M events/sec sustained, average payload 500 bytes (~2.5 GB/sec), bursting up to 3x baseline.
- **Latency:** ingestion-to-availability p99 < 10 s.
- **Delivery:** at-least-once; ability to replay the last 7 days of events for reprocessing.
- **Correctness:** windowed aggregates must account for events that arrive after the window they belong to has "ended," within a bounded grace period.

**Capacity check:** 2.5 GB/sec sustained is squarely in the range transport layers like `DistributedMessageQueue.md` are built for; the genuinely hard part of this problem isn't moving the bytes, it's computing *correct* time-windowed aggregates when events don't arrive in the order they happened.

## 3. Interface

```
PRODUCE stream={s} event={payload, event_time} -> ack
SUBSCRIBE stream={s} from={offset|replay_timestamp} -> events[]
```

## 4. High-Level Architecture — `RealTimeDataIngestionPipeline_component.excalidraw`

- **Producers → Ingestion Log**: durable, ordered-per-partition transport (same leader/replica model as `DistributedMessageQueue.md` §5).
- **Stream Processor → Ingestion Log**: consumes events, tracks the watermark (§5), and maintains open windows.
- **Stream Processor → Window Store**: holds in-flight (not-yet-finalized) window aggregates, keyed by window + partition key.
- **Stream Processor → Output Sink**: emits finalized (and, if needed, corrected) window results downstream.
- **Late Event Sink**: events arriving after a window's allowed-lateness grace period are routed here instead of silently dropped.

## 5. Deep Dive — Event-Time Windowing & Watermarks — `RealTimeDataIngestionPipeline_sequence.excalidraw`

**Options considered:**
- Group events by *processing time* (when the system saw them), ignoring any timestamp embedded in the event itself — simple, but a late-arriving event (delayed by network or client buffering) lands in whatever window happens to be open at arrival time, attributing it to the wrong time bucket and skewing aggregates.
- **Event-time windowing with watermarks — the pick:** each event carries its own event-time timestamp; the system tracks a watermark — an estimate of "we've seen everything up to time T" — and only finalizes a window once the watermark passes its end, holding it open briefly to absorb a bounded amount of lateness.

**Terms:**

| Term | Meaning |
|---|---|
| Event time | When the event actually happened, per its own embedded timestamp. |
| Processing time | When the system observed/received the event — not used for window assignment. |
| Watermark | The system's running estimate of "no more events older than T will arrive." |
| Allowed lateness | A grace period after the watermark passes a window's end during which late events can still update it, instead of being dropped. |

**Worked timeline** (1-minute window `[12:00:00, 12:01:00)`, 1-minute allowed lateness):

| Time | Event | Window state |
|---|---|---|
| 12:01:00 | Watermark reaches 12:01:00 | Window finalized and emitted using events seen so far |
| 12:01:15 | Late event, event_time=12:00:45, arrives (30s transit delay) | Within allowed lateness → window reopened, corrected aggregate re-emitted |
| 12:02:30 | Another late event, event_time=12:00:10, arrives | Past the allowed-lateness grace period → routed to the Late Event Sink, not silently dropped |

**Correctness pitfall:** setting allowed lateness to zero (finalizing strictly at the watermark with no grace period) silently discards every late event's contribution with no visibility into how much data was lost. The drop itself must be an explicit, monitored decision — a `late_events_dropped` metric, not a silent default — otherwise aggregates quietly diverge from reality with no signal that anything is wrong.

## 6. Data Model

- Window store entry: `window_key (stream, partition_key, window_start), aggregate_state, last_updated_watermark`.
- Watermark state: `partition -> current_watermark`, advanced as events are consumed.
- Late event record: `event, original_window, arrival_time` — routed separately for manual/batch reconciliation.

## 7. Scaling, Failure & Trade-offs

- Watermarks are tracked per-partition and combined (taking the minimum across partitions feeding a window) — a single slow partition holds back finalization for any window it contributes to, a deliberate correctness-over-latency choice.
- A stream processor crash loses only in-memory window state, not correctness: replaying from the last committed offset in the Ingestion Log rebuilds the same window state deterministically, the same recovery principle as `WorkflowOrchestrationEngine.md` §5's replay-based durability.
- Allowed lateness is a tunable trade-off: longer grace periods catch more late data but delay when results are considered "final" and increase the memory cost of keeping windows open.

## 8. Wrap-up

Delivered: an event-time windowing design with watermarks and a bounded, monitored lateness grace period. Not covered due to time: schema evolution, exactly-once sink semantics, and the specific aggregation logic beyond windowing mechanics.