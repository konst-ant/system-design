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
- **Suggestion Service → Suggestion Store**: returns the precomputed Top-K list for that prefix (§5) — implemented as either an in-memory Trie or a Redis Sorted Set (§5a/§5b).
- **Query Log Aggregator (offline, hourly)**: scans recent search logs, recomputes popularity counts, rebuilds the affected Top-K entries.
- **Store Builder**: publishes the refreshed Top-K data to all Suggestion Service replicas (Trie snapshot) or into Redis (ZSET rebuild).

## 5. Deep Dive — Precomputed Top-K Autocomplete — `SearchAutocompleteSystem_sequence.excalidraw`

**Shared principle, regardless of implementation:**
- Live scan, rank on every query by walking/scanning all completions under the prefix and sorting by frequency — correct and always fresh, but for a short/common prefix the candidate set can be enormous, making per-request ranking far too slow for a 50 ms budget.
- **Precomputed Top-K, computed offline — the pick:** popularity ranking is done once, offline; a query just reads what's already ranked — cost stays small and bounded regardless of how popular a prefix is. Two viable ways to store and serve this are below (§5a, §5b) — same principle, different infrastructure.

**Shared terms:**

| Term | Meaning |
|---|---|
| Precomputed Top-K | The cached list of a prefix's K most popular completions, computed offline. |
| Query log aggregation | Offline job that counts completed searches over a recent window and recomputes popularity. |
| Refresh cycle | How often the aggregation job runs and republishes updated Top-K lists (hourly here). |
| Staleness bound | The maximum time a genuinely popular new query can remain invisible — bounded by the refresh cycle, not unbounded. |

**Shared correctness pitfall:** a viral new query won't appear in any Top-K list until the next hourly run — if near-real-time trending suggestions are required, the fix is a *separate*, small, frequently-refreshed "trending" overlay merged in front of the precomputed list, not shortening the main refresh cycle or computing live on the read path — either of those reintroduces the exact scan cost this design exists to avoid.

### 5a. Implementation Option 1 — In-Memory Trie

Each prefix is a trie node; the node stores its own already-sorted Top-K directly, so a query is just a walk of `prefix length` hops.

**Worked timeline:**

| Time | Event | Node "ap"'s cached Top-K |
|---|---|---|
| t=0 | Hourly aggregation job runs, recomputes from the last hour's logs | `["apple", "apply", "app"]` |
| t=0–60min | User types "ap" → Suggestion Service walks 2 trie hops, returns cached list directly, no scanning | unchanged |
| t=60min | Next aggregation run picks up a newly-trending query | `["apple", "apex", "apply"]` — "apex" now appears |

**Trade-off:** zero network cost (in-process lookup), but it's a custom data structure — you build, test, and maintain the trie and its atomic-snapshot-swap/replication logic yourself.

### 5b. Implementation Option 2 — Redis Sorted Set (`ZRANGEBYLEX`)

Every phrase is stored **once**, as a single ZSET member, score = 0 for all members (required for `ZRANGEBYLEX` to use pure lexicographic ordering): `word\x00rank_padded`. A query for prefix `"ap"` runs `ZRANGEBYLEX key "[ap" "[ap\xff"`, returning every match — but for a **short, high-fan-out prefix** that's the entire matching subtree, which is too large to sort live. So the same offline job additionally precomputes and caches the Top-K for prefixes below a length cutoff (e.g. ≤ 3–4 characters); only prefixes *above* the cutoff — naturally small result sets — are safely ranked live from the full scan.

**Worked example** (prefix `"ap"`, already rank-tagged):

| Member stored | Lex position for prefix "ap" |
|---|---|
| `apple\x0001` | 1st |
| `apply\x0002` | 2nd |
| `app\x0003` | 3rd |

**Correctness pitfall:** skipping the length cutoff and live-scanning every prefix reintroduces the exact subtree-blowup problem this whole design exists to avoid — short/common prefixes are the most frequently typed, not an edge case to deprioritize.

**What's attractive here:** this is an off-the-shelf Redis primitive, not bespoke code — no custom data structure to design, build, or debug. Replication, sharding, and durability come from Redis itself (battle-tested, already operated at scale), rather than being hand-rolled snapshot/replication logic for a custom trie.

### 5c. Choosing Between Them

| | Trie (in-process) | Redis ZSET |
|---|---|---|
| Memory | Shares storage across common prefixes — smaller footprint | Each phrase stored once, no prefix-sharing, but no per-prefix duplication either |
| Latency | Zero network hop | One Redis round-trip (sub-ms, but non-zero) |
| Build/ops effort | Custom structure: you build, test, and operate snapshot/replication logic | Off-the-shelf: replication, sharding, durability provided by Redis |

Either is a reasonable "win" for this problem; Redis trades a small, predictable latency cost for not having to build and operate custom infrastructure.

## 6. Data Model

- Trie node (Option 1): `children map, cached_top_k: [(phrase, score)]`.
- Redis ZSET (Option 2): members `word\x00rank_padded`, score 0; short-prefix Top-K cache as a separate small key per prefix.
- Query log aggregate (offline, shared by both): `phrase -> count_in_window`, rolled up during the build.

## 7. Scaling, Failure & Trade-offs

- Both options are read-only on the hot path — scaling reads is just adding replicas (trie replicas or Redis read replicas).
- A new Top-K snapshot is published atomically in both cases (trie: swap the whole in-memory structure; Redis: build into a fresh key, then `RENAME`) — never a window where some entries reflect old ranking and others new.
- Regional/language variants are handled as entirely separate stores selected by the client's locale, rather than baking a language dimension into every entry.

## 8. Wrap-up

Delivered: two viable implementations of the same O(bounded-cost) precomputed-Top-K principle — a custom in-memory Trie, and an off-the-shelf Redis Sorted Set — with an explicit, bounded staleness trade-off for freshness in both. Not covered due to time: personalization, fuzzy/typo matching, and a trending-query overlay.