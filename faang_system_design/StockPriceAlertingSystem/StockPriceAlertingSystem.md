# Stock Price Alerting System — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `StockPriceAlertingSystem_component.excalidraw`, `StockPriceAlertingSystem_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform that evaluates tens of millions of user-defined price alert rules against a high-frequency market data stream, guaranteeing no genuine threshold-crossing is ever silently missed.

**In scope:** rule indexing, reactive (event-triggered) evaluation, backpressure under volume spikes.

**Explicitly deferred:** watchlist UI, complex multi-condition rule types, notification delivery (see `NewsFeedSystem.md`'s fan-out shape for the general delivery pattern).

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~1M price ticks/sec ingested during trading hours; ~50M user-defined alert rules.
- **Latency:** tick arrival → notification dispatch p99 < 1 s.
- **Correctness:** no alert may be silently missed, even during volume spikes at market open/close.

**Capacity check:** scanning all 50M rules on every one of 1M ticks/sec is obviously infeasible — the vast majority of rules watch a symbol that isn't the one that just ticked, so the real design problem is routing each tick only to the (typically small) set of rules that actually care about it.

## 3. Interface

```
POST /alerts {symbol, condition, threshold} -> {alert_id}
Tick stream (internal): {symbol, price, timestamp} -> triggers reactive evaluation
```

## 4. High-Level Architecture — `StockPriceAlertingSystem_component.excalidraw`

- **Market Data Feed → Tick Ingestion**: durable, buffered ingestion of price ticks (same buffering principle as `LoggingEventCollectionPlatform.md` §5).
- **Tick Ingestion → Rule Registry**: looks up which rules watch the ticked symbol (§5).
- **Rule Registry → Evaluator**: checks only the relevant rules' conditions against the new price.
- **Evaluator → Notification Dispatcher**: fires for any rule whose condition is now met.

## 5. Deep Dive — Symbol-Indexed Reactive Evaluation — `StockPriceAlertingSystem_sequence.excalidraw`

**Options considered:**
- Periodic polling: every N seconds, scan all 50M rules, fetch each rule's symbol's latest price, check the condition — simple, but introduces up to N seconds of detection delay regardless of actual tick rate, and wastes work re-checking rules for symbols that haven't changed since the last poll.
- **Reactive, symbol-indexed rule registry — the pick:** rules are indexed as `symbol -> [rules watching this symbol]`. When a tick arrives for symbol X, only the rules registered against X are evaluated — immediately, triggered by the tick itself, not a scheduled scan.

**Terms:**

| Term | Meaning |
|---|---|
| Rule registry | Index: `symbol -> [active alert rules for that symbol]`. |
| Tick | A single price update event for one symbol — the unit that triggers evaluation. |
| Reactive evaluation | Checking only the rules relevant to the symbol that just changed, triggered by the tick, not a periodic scan. |
| Debounce | An already-triggered rule that would re-fire on every subsequent tick while still true is suppressed until it resets, avoiding a notification storm from one sustained price move. |

**Worked timeline:**

| Time | Event | Rules touched | Outcome |
|---|---|---|---|
| t=0 | Tick arrives: AAPL = $150.00 | 3,000 rules watching AAPL (out of 50M total) | the other ~49,997,000 rules are never touched |
| t=0 | Each of the 3,000 evaluated against $150.00 | 12 match their threshold | notifications dispatched, all within the 1s budget |

**Correctness pitfall:** if the registry lookup or evaluator is allowed to silently drop ticks under load (e.g. discarding ticks to "catch up" when backlogged), a genuine threshold-crossing tick could be exactly the one dropped — violating the "no alert may be silently missed" NFR outright. The system must apply backpressure — buffer and queue ticks, the same durable-buffering principle as `LoggingEventCollectionPlatform.md` §5 — rather than drop, even under the volume spike at market open/close.

## 6. Data Model

- Rule registry: `symbol -> [{alert_id, condition, threshold, last_triggered_at}]`.
- Tick buffer: durable queue absorbing bursts, same role as `LoggingEventCollectionPlatform.md`'s agent buffer, sized for open/close volume spikes rather than average throughput.
- Alert: `alert_id, user_id, symbol, condition, threshold, status (active|triggered|debounced)`.

## 7. Scaling, Failure & Trade-offs

- The registry is sharded by symbol, so evaluation for different symbols scales embarrassingly in parallel — no symbol's rule set needs to coordinate with any other's.
- A popular symbol (millions of watching rules, e.g. a major index) is the one place contention can concentrate — the same hot-partition handling as `WebCrawler.md` §7 and `MessageStorageSystem.md` §7, here applied to "rules watching one symbol" instead of "a domain's URLs" or "one conversation's messages."
- Debouncing trades a small risk of missing a very-short-lived re-cross for protection against spamming a user with a notification on every tick during a sustained move through their threshold — an explicit, tunable choice, not a correctness gap, since the first crossing is never missed.

## 8. Wrap-up

Delivered: a symbol-indexed, reactive evaluation design that avoids scanning irrelevant rules and applies backpressure instead of dropping ticks under load. Not covered due to time: watchlist UI, complex multi-condition rules, and notification delivery mechanics.