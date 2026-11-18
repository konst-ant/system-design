# Search Engine — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `SearchEngine_component.excalidraw`, `SearchEngine_sequence.excalidraw`.*

## 1. Problem & Scope

Design a system that indexes a web-scale document corpus and answers keyword queries with ranked results, with continuous ingestion of new/updated documents.

**In scope:** inverted index design, index build/update pipeline, query serving/ranking path, sharding strategy.

**Explicitly deferred:** the crawler itself (covered by a separate exercise), ML-based ranking model internals, query understanding (spell-correction, synonyms), spam/SEO-abuse detection. These are each their own 45-minute problem; I'll note where they'd plug in.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~10B documents indexed; ~100M documents added/updated per day (~1,160 docs/sec sustained ingest).
- **Query load:** ~200K queries/sec at global peak.
- **Latency:** p99 query latency < 200 ms end-to-end.
- **Freshness:** index staleness of a few hours is acceptable — this is not a strongly consistent system.

**Index size estimate:** assume ~1 KB of compressed postings data per document on average (term IDs, positions, frequencies). 10B docs × 1 KB ≈ 10 TB raw index. Sharded across ~1,000 shards → ~10 GB/shard; replicated 3× for availability and read scaling → ~30 TB total disk, which comfortably fits a commodity SSD fleet.

**Sharding:** document-partitioned with scatter-gather across ~1,000 shards — the standard choice at this scale for even load distribution, paired with aggressive caching to absorb the Zipfian query distribution. Trade-off and mechanics in §5b.

## 3. Interface

```
GET /search?q=<query>&page=<n> -> { results: [{doc_id, title, snippet, score}], total_estimate }
```

## 4. High-Level Architecture — `SearchEngine_component.excalidraw`

**Ingestion path:** Crawler/Feed → Raw Document Store (blob storage) → Indexing Pipeline (parse, tokenize, analyze, extract ranking signals) → new immutable index segments written to the relevant shards.

**Query path:** Client → Query Router/LB → Query Cache (hot/repeat queries) → on cache miss: Query Service fans out in parallel to index shards → each shard returns its local top-K candidates + partial score → Query Service merges results → Ranking Service re-scores the merged top candidates with richer signals → response.

## 5. Deep Dive — `SearchEngine_sequence.excalidraw`

### 5a. Index Update Strategy

**Options considered:**
- Full rebuild — simple, but can't run continuously at 100M docs/day; staleness and cost are unacceptable.
- In-place mutation — fast to apply, but concurrent readers can see a torn/inconsistent index mid-write; needs heavy locking.
- **LSM-style immutable segments + background merge — the pick:** writes never block reads; updates/deletes go through tombstones, not in-place edits; staleness is bounded by how often new segments commit.

**Terms:**

| Term | Meaning |
|---|---|
| Segment | Immutable chunk of the inverted index, built once from a batch of docs, never modified after it's committed. |
| Tombstone | Marker recording that a doc_id is deleted/superseded — doesn't touch the old segment's bytes. |
| Compaction (merge) | Background job combining several segments into one, physically dropping tombstoned data. |
| Reference counting | Tracks which in-flight queries still depend on an old segment, so it isn't deleted out from under them. |

**Worked timeline:**

| Time | Event | On-disk segments | What a query sees |
|---|---|---|---|
| t=0 | Segment S1 committed (doc X included) | S1 | X's content from S1 |
| t=1 | Doc X updated → new segment S2 committed; X tombstoned in S1 | S1(tombstone X), S2 | X's content from S2 |
| t=2 | Compaction merges S1+S2 → S3; tombstoned bytes physically dropped | S3 | X's content from S3 (same logical result) |

**Correctness pitfall:** a long-running query that opened S1+S2 at t=1.9 must keep reading those exact files even after compaction produces S3 at t=2 — otherwise a doc could vanish mid-query. Fix: segments are only physically deleted once no in-flight query still references them (reference counting / epoch-based reclamation), not immediately after compaction.

### 5b. Query Execution Strategy

**Options considered:**
- Term-partitioned (global index) — a query only touches shards owning its terms, but creates hot shards for common terms and complex multi-term joins.
- **Document-partitioned + scatter-gather — the pick:** every query fans out to all shards, but load balances naturally since documents are spread uniformly — the standard choice at this scale.

**Terms:**

| Term | Meaning |
|---|---|
| Shard | One partition of documents and its own local inverted index. |
| Local top-K | Each shard's own best-K candidates, cheaply scored (e.g. BM25), returned independently. |
| Per-shard timeout | Bounded wait per shard; a shard that doesn't answer in time is excluded, not blocking the query. |
| Merge (reduce) | Combining all shards' local top-K lists into one global candidate set by score. |
| Best-effort result | A response built from whichever shards answered in time — fewer candidates, bounded latency. |

**Worked timeline** (100 ms per-shard timeout, 3 shards):

| Time | Shard 1 | Shard 2 | Shard 3 |
|---|---|---|---|
| t=0ms | dispatched | dispatched | dispatched |
| t=40ms | returned top-K | pending | pending |
| t=60ms | — | returned top-K | pending |
| t=100ms | — | — | timed out → excluded |

Merge proceeds with Shard 1 + 2 only; the response is marked best-effort/partial rather than complete.

**Correctness pitfall:** a partial result must be explicitly flagged (e.g. `partial: true`, adjusted `total_estimate`) — silently returning it as if every shard answered would corrupt pagination and make recall-quality regressions invisible to monitoring.

## 6. Data Model

- **Per shard:** inverted index — `term → sorted postings list of (doc_id, term_freq, positions)`; forward index/doc store for snippet generation; doc metadata (last-modified, static popularity signal) used by ranking.
- **Segment files:** immutable, versioned, with tombstone markers for deleted/updated docs.

## 7. Scaling, Failure & Trade-offs

- Replicas per shard both absorb read load and provide failover; the router picks a healthy, least-loaded replica.
- Query cache absorbs the bulk of repeat traffic given a Zipfian query distribution — cheapest latency win available.
- Freshness vs. cost trade-off behind the indexing strategy: see §5a.
- Consistency is intentionally eventual: a newly indexed doc may take minutes to show up in results, which matches the stated freshness target.

## 8. Wrap-up

Delivered: a sharded, continuously-updated inverted index with a scatter-gather query path and an explicit availability/recall trade-off under partial shard failure. Not covered due to time: ML re-ranking internals, query understanding (spell-correct/synonyms), and spam/abuse detection — each would be a natural follow-up deep dive.
