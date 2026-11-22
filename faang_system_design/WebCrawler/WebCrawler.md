# Web Crawler — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `WebCrawler_component.excalidraw`, `WebCrawler_sequence.excalidraw`.*

## 1. Problem & Scope

Design a distributed crawler that discovers and downloads web pages at scale, without overloading any single target site.

**In scope:** URL frontier management, deduplication, politeness enforcement, distributed worker coordination.

**Explicitly deferred:** content extraction/parsing quality, robots.txt rule-interpretation edge cases, ML-based crawl prioritization. These sit downstream of the core frontier/politeness design.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** frontier of ~10B URLs; sustained crawl rate ~50K pages/sec across tens of thousands of worker instances.
- **Politeness:** ≤1 request/sec per domain, regardless of how many workers exist.
- **Freshness:** high-priority sites re-crawled within 24 hours.
- **Availability:** worker failures must not cause large-scale redundant re-crawling or runaway load on any single site.

**Capacity check:** 50K pages/sec across, say, 50,000 domains averaging 1 req/sec/domain each would exactly saturate the politeness budget — in practice traffic is concentrated on far fewer domains at any moment, so the real constraint is ensuring *no single domain* exceeds its 1 req/sec cap even while the fleet overall pushes 50K req/sec in aggregate.

## 3. Interface

Internal system, no public API — the unit of work is a frontier entry:

```
Frontier entry: {url, domain, priority, next_allowed_fetch_time}
```

## 4. High-Level Architecture — `WebCrawler_component.excalidraw`

- **URL Frontier**: durable queue of URLs to fetch, partitioned by domain (§5).
- **Fetcher Workers**: pull URLs from their assigned domain shard, fetch content, respect `next_allowed_fetch_time`.
- **Dedup Store**: bloom filter / hash set of already-seen URLs, checked before adding a new URL to the frontier.
- **Parser**: extracts new links from fetched content, feeds them back to the Frontier (after dedup check).
- **Content Store**: durable storage for fetched pages.
- **Domain Timer Store**: shared, authoritative `next_allowed_fetch_time` per domain (§5).

## 5. Deep Dive — Politeness Enforcement — `WebCrawler_sequence.excalidraw`

**Options considered:**
- Single global queue, any worker pulls any URL — simple to load-balance, but nothing stops two different workers from independently picking URLs for the same domain at the same moment, violating the politeness limit.
- **Domain-sharded queues + shared per-domain timer — the pick:** URLs are partitioned by `hash(domain)` so all URLs for one domain route to the same shard; a worker must check and update that domain's `next_allowed_fetch_time` in a shared store before fetching. A Redis Sorted Set (score = `next_allowed_fetch_time`, member = domain) is a good off-the-shelf fit — `ZRANGEBYSCORE`/`ZADD` give an atomic, shared "pop whichever domain is ready" primitive without building a custom heap.

**Terms:**

| Term | Meaning |
|---|---|
| Domain shard | Partition of the frontier holding all URLs for a set of domains — ensures one domain is never split across workers that don't coordinate. One domain maps to exactly one shard, but many domains map to that same shard; many workers can still operate concurrently within it. |
| Next-allowed-fetch-time | Shared timestamp per domain: the earliest moment any worker may fetch from it again. |
| Not due yet (skip) | If a domain's score (`next_allowed_fetch_time`) hasn't elapsed, the worker just leaves it in the priority set and checks the next-ready domain instead — no dequeue/requeue step, nothing moves. |
| Back-off | Additional delay added to a domain's timer after receiving a 429/503 from that site. |

**Worked timeline** (politeness = 1 req/sec/domain):

| Time | Event | Domain X's shared timer |
|---|---|---|
| t=0.0s | Worker 1 fetches a URL for domain X | sets next-allowed = t=1.0s |
| t=0.3s | Worker 2 checks domain X's score in the priority set | 1.0s not yet reached → **not due yet**, skips it and checks the next-ready domain instead, does **not** fetch |
| t=1.0s | Worker 3 pulls a URL for domain X, checks shared timer | 1.0s reached → fetches, sets next-allowed = t=2.0s |

**Correctness pitfall:** the timer must live in a store every worker can see — if each worker instead cached its own *local* "last fetched domain X at..." value, two workers with no visibility into each other's activity could both decide it's safe to fetch simultaneously, exactly violating the 1 req/sec limit this design exists to enforce. (The same lesson as the Rate Limiter's local-cache-vs-source-of-truth distinction — here there's no local-cache fast path at all, because a domain is only ever handled by the one shard that owns it, so the "shared store" check is cheap enough to do on every fetch.)

## 6. Data Model

- Frontier entry: `url, domain, priority, next_allowed_fetch_time, discovered_at`.
- Dedup store: `url_hash -> seen` (bloom filter for the common case, backed by an exact set for high-priority URLs where false positives are costlier). 
- Domain timer store: `domain -> next_allowed_fetch_time, consecutive_failure_count` (drives back-off) — e.g. a Redis ZSET per shard, scored by `next_allowed_fetch_time`. A Kafka topic upstream of this (fed by the Parser) may be needed to buffer bursts of newly-discovered URLs before they reach a shard.

## 7. Scaling, Failure & Trade-offs

- A crashed worker simply drops its in-flight URL, which times out and is re-queued — no special recovery needed, since the frontier is durable and URLs aren't removed until successfully fetched.
- Domain sharding naturally load-balances across workers *as long as* no single domain is disproportionately huge; a domain with an unusually large URL count gets its own dedicated worker pool rather than starving others on the same shard.
- Bloom filter false positives mean a small fraction of genuinely-new URLs get silently skipped — an accepted trade-off at this scale, since the alternative (exact dedup at 10B URLs) costs far more storage for a marginal recall gain.

## 8. Wrap-up

Delivered: a domain-sharded frontier with a shared politeness timer that scales to tens of thousands of workers without any single domain being overloaded. Not covered due to time: robots.txt parsing nuances, content-quality-based prioritization, and distributed dedup store partitioning details.