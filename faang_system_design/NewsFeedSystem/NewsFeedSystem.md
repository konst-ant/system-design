# News Feed System — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `NewsFeedSystem_component.excalidraw`, `NewsFeedSystem_sequence.excalidraw`.*

## 1. Problem & Scope

Design a system that lets users publish posts and generates a personalized feed aggregating posts from the entities they follow.

**In scope:** publish path, feed generation/fan-out strategy, feed read path, handling highly skewed follow graphs ("celebrity" accounts).

**Explicitly deferred:** the ML ranking model itself (we use a simple recency+engagement heuristic and note where a model would plug in), likes/comments and their own counters/fan-out, media storage/CDN delivery, ads insertion. Each is a separate system worth its own session.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~500M DAU; ~100M posts/day (~1,160 posts/sec average, bursty); highly skewed follow graph — most accounts have a few hundred followers, but some have 50M+ ("celebrities").
- **Read load:** ~5B feed-read requests/day → ~58K RPS average, ~180K RPS peak.
- **Latency:** p99 feed-read < 200 ms.
- **Freshness:** a new post should reach followers' feeds within seconds to low minutes — eventual consistency is fine.

**The calculation that drives the whole design:** average user has ~300 followers. Posting at 1,160/sec × 300 followers ≈ 350K feed-cache writes/sec from fan-out alone for *average* users — manageable with async workers behind a queue. But a single post from a 50M-follower celebrity, fanned out the same way, is **50 million writes for one post**. That asymmetry — not the average case — is the actual design problem here, and it's why a pure "push to everyone" design doesn't work at this scale.

## 3. Interface

```
POST /posts {content} -> {post_id}
GET  /feed?cursor=<cursor> -> {posts: [...], next_cursor}
```

## 4. High-Level Architecture — `NewsFeedSystem_component.excalidraw`

**Write path:** Client → Post Service → Post Store (durable, sharded by `author_id`) → publish a `PostCreated` event to Kafka → Fan-out Worker consumes the event: if the author's follower count is below a threshold (e.g. 10K), push the `post_id` into every follower's precomputed feed (Redis ZSET, capped length ~1,000). If the author is a "celebrity" (above threshold), **skip push entirely**.

**Read path:** Client → Feed Service → read the precomputed feed from Feed Cache (fast path, covers everyone the user follows who isn't a celebrity) → separately pull the latest posts from the small number of celebrities the user follows directly from the Post Store → merge the two lists by recency/score → paginate → return.

## 5. Deep Dive — `NewsFeedSystem_sequence.excalidraw`

**Options considered** (cost driven by the math in §2):
- Fan-out-on-write (push) — O(1) read, O(followers) write; breaks down for celebrities (50M writes for one post).
- Fan-out-on-read (pull) — O(1) write, O(follows) read; fine per-request, but applied universally it makes every feed read scan every followee live.
- **Hybrid — the pick:** push for accounts below a follower-count threshold; pull-and-merge at read time for the few celebrities above it.

**Terms:**

| Term | Meaning |
|---|---|
| Feed cache | Per-user precomputed list of post_ids (Redis ZSET) — populated only by push. |
| Celebrity threshold | Follower-count cutoff above which an account is excluded from push fan-out. |
| Fan-out worker | Async consumer that, on a new post, pushes post_id into every non-celebrity follower's feed cache. |
| Celebrity pull | At read time, fetch latest posts directly from each followed celebrity — cheap, since a user follows few celebrities. |
| Merge-at-read | Combine the precomputed pushed feed with freshly pulled celebrity posts, sort by recency, return. |

**Worked timeline:**

| Time | Event | Fan-out? | Follower's feed cache | Read-time merge |
|---|---|---|---|---|
| t=0 | Normal user (300 followers) posts | Yes → pushed to 300 feed caches | post_id appended | — |
| t=0 | Celebrity (50M followers) posts | No → skipped entirely | unchanged | — |
| t=5s | A mutual follower opens their feed | n/a | already has the normal user's post | + pulls the celebrity's latest posts live, merges by recency, returns |

**Correctness pitfall:** the celebrity threshold must be evaluated once, atomically, at fan-out time using a consistent follower-count snapshot — not re-checked independently at read time. Otherwise an account crossing the threshold mid-flight (e.g. going viral) could be double-counted (pushed *and* pulled) or missed (neither), instead of being cleanly handed off from one path to the other.

## 6. Data Model

- **Post store:** `post_id, author_id, content_ref, created_at` — sharded by `author_id` (keeps an author's posts co-located for the celebrity-pull path).
- **Feed cache:** Redis ZSET per `user_id`; score = recency/rank, member = `post_id`; capped at ~1,000 entries to bound memory.
- **Follow graph:** `followee_id → [follower_ids]` (partitioned by followee, used by the fan-out worker) and `follower_id → [followee_ids]` (used at read time to find which followees are celebrities).

## 7. Scaling, Failure & Trade-offs

- Fan-out workers scale horizontally behind Kafka partitions; the "celebrity" threshold is a tunable config value, re-evaluated periodically as accounts grow.
- **Write/fan-out decoupling:** the post is durably written to the Post Store *before* fan-out happens, so a slow or temporarily down fan-out worker only delays propagation (within the freshness budget) — it never blocks or loses the write.
- **Cold feed cache** (new user, or cache evicted): fall back to an on-demand pull-and-merge across *all* of the user's follows — slower, but rare, so it doesn't need to be fast.
- Consistency is intentionally eventual throughout — a follower might see a post a few seconds late; this matches the stated freshness NFR and isn't a bug to engineer away.

## 8. Wrap-up

Delivered: a hybrid push/pull fan-out design justified by an explicit capacity calculation, plus the merge-at-read step that makes it work. Not covered due to time: a real ranking model beyond the recency heuristic, like/comment counters and their own propagation, and media/ads delivery.
