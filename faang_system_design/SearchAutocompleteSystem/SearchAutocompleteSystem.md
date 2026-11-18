# Search Autocomplete System — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `SearchAutocompleteSystem_component.excalidraw`, `SearchAutocompleteSystem_sequence.excalidraw`.*

## 1. Problem & Scope

Design a service that suggests completions while a user types into a search box, ranked by popularity.

**In scope:** the prefix-matching data structure, the offline popularity-ranking update cycle, the online read path.

**Explicitly deferred:** personalization, typo-tolerance/fuzzy matching, locale-specific tokenization. Each would layer on top of this core design without changing its shape.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~100M DAU; each firing roughly 15–20 keystroke-triggered requests/day → ~20K RPS average, ~60K RPS peak.
- **Latency:** p99 < 50 ms server-side — this fires on every keystroke, so it must feel instant.
- **Freshness:** suggestion popularity refreshed hourly from aggregated query logs; a brand-new trending query is invisible until the next refresh — an accepted staleness bound (§5).
- **Data size:** suggestion index covers ~50M distinct phrases.

**Capacity check:** 60K RPS of prefix lookups, each just a few pointer hops through a trie, is CPU-cheap per request — the real constraint is keeping the whole structure in memory across a modest fleet (a 50M-phrase trie with small per-node metadata is on the order of a few GB, well within a single large instance, replicated for availability/read-scaling).

## 3. Interface

```
GET /suggest?prefix=<text> -> { suggestions: [string, ...] }  // top-K, pre-ranked
```

## 4. High-Level Architecture — `SearchAutocompleteSystem_component.excalidraw`

- **Client → Suggestion Service**: on every keystroke, sends the current prefix.
- **Suggestion Service → Trie (in-memory)**: walks to the prefix's node, returns its precomputed Top-K list (§5).
- **Query Log Aggregator (offline, hourly)**: scans recent search logs, recomputes popularity counts, rebuilds each affected trie node's Top-K.
- **Trie Builder**: publishes a new trie snapshot to all Suggestion Service replicas.

## 5. Deep Dive — Precomputed Top-K per Trie Node — `SearchAutocompleteSystem_sequence.excalidraw`

**Options considered:**
- Live trie, rank on every query by walking the full subtree under the prefix and sorting by frequency — correct and always fresh, but for a short/common prefix the subtree can be enormous, making per-request ranking far too slow for a 50 ms budget.
- **Trie with precomputed Top-K cached per node — the pick:** popularity ranking is done once, offline, and each node stores only its own already-sorted Top-K list; a query just walks to the node and returns what's already there — cost is O(prefix length), not O(subtree size).

**Terms:**

| Term | Meaning |
|---|---|
| Trie node | One prefix's position in the tree; has children for each possible next character. |
| Precomputed Top-K | A node's cached list of its K most popular completions, computed offline. |
| Query log aggregation | Offline job that counts completed searches over a recent window and recomputes popularity. |
| Refresh cycle | How often the aggregation job runs and republishes updated Top-K lists (hourly here). |
| Staleness bound | The maximum time a genuinely popular new query can remain invisible — bounded by the refresh cycle, not unbounded. |

**Worked timeline:**

| Time | Event | Node "ap"'s cached Top-K |
|---|---|---|
| t=0 | Hourly aggregation job runs, recomputes from the last hour's logs | `["apple", "apply", "app"]` |
| t=0–60min | User types "ap" → Suggestion Service walks 2 trie hops, returns cached list directly, no scanning | unchanged |
| t=60min | Next aggregation run picks up a newly-trending query | `["apple", "apex", "apply"]` — "apex" now appears |

**Correctness pitfall:** a viral new query won't appear in any node's Top-K until the next hourly run — if a product requirement demands near-real-time trending suggestions, the right fix is a *separate*, small, frequently-refreshed "trending" overlay merged in front of the precomputed list, not shortening the main refresh cycle or recomputing synchronously on the read path — either of those would reintroduce the exact subtree-scan cost this design exists to avoid.

## 6. Data Model

- Trie node: `children map, cached_top_k: [(phrase, score)]`.
- Query log aggregate (offline): `phrase -> count_in_window`, rolled up to the phrase's ancestor nodes during the build.

## 7. Scaling, Failure & Trade-offs

- The trie is read-only on the hot path — replicated identically across Suggestion Service instances, so scaling reads is just adding replicas.
- A new trie snapshot is published atomically (swap the whole in-memory structure), so there's never a window where some nodes reflect the old ranking and others the new one.
- Regional/language variants are handled as entirely separate tries selected by the client's locale, rather than one trie with a language dimension baked into every node.

## 8. Wrap-up

Delivered: an O(prefix length) read path via precomputed per-node rankings, with an explicit, bounded staleness trade-off for freshness. Not covered due to time: personalization, fuzzy/typo matching, and a trending-query overlay.