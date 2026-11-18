# Social Media Platform — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `SocialMediaPlatform_component.excalidraw`, `SocialMediaPlatform_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform where users create profiles, publish posts, follow others, and engage via likes/comments/shares.

**In scope:** post/profile storage, follow relationships, media attachment references, and — as the deep dive — engagement counters under heavy concurrency.

**Explicitly deferred:** feed ranking/fan-out (its own exercise — see `NewsFeedSystem.md`), abuse/spam detection, comment threading structure. These layer on top without changing the post/counter design.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~300M DAU; ~50M posts/day; write:read ratio roughly 1:100 (posts are rare relative to views/engagements).
- **Latency:** post-creation p99 < 500 ms; read (profile/post) p99 < 200 ms.
- **Availability:** 99.9%.
- **Consistency:** engagement counters (likes, follower counts) may be eventually consistent; post content and media must never be lost.

**Capacity check:** a single post going viral can receive thousands of likes/sec. A naive `UPDATE posts SET like_count = like_count + 1 WHERE id = X` serializes every one of those increments on the same database row — at even 1,000 likes/sec that's 1,000 row-lock acquisitions/sec on one row, far beyond what a relational row lock comfortably sustains alongside normal read traffic on the same table.

## 3. Interface

```
POST /posts {content} -> {post_id}
POST /posts/{id}/like  (idempotent per user)
GET  /posts/{id} -> {content, like_count, comment_count, ...}
```

## 4. High-Level Architecture — `SocialMediaPlatform_component.excalidraw`

- **Client → Post Service**: create/read posts, durable write to Post Store.
- **Client → Engagement Service**: like/unlike, comment — hot path described in §5.
- **Engagement Service → Hot Counter Store (Redis)**: atomic in-memory increment, sub-millisecond.
- **Flush Worker**: periodically reconciles Redis counters into the durable Post Store.
- **Post Store**: durable record of posts, profiles, and the authoritative (slightly-lagged) counter values.

## 5. Deep Dive — Engagement Counters Under Concurrency — `SocialMediaPlatform_sequence.excalidraw`

**Options considered:**
- Direct DB increment on every like (`UPDATE ... SET like_count = like_count + 1`) — correct, but a viral post turns into a hot row: every like serializes on the same row lock, and that row sits in the same table as normal read traffic, dragging down unrelated queries too.
- **Redis atomic counter + async write-behind flush — the pick:** likes increment an in-memory Redis counter (`INCR`, sub-ms, lock-free at this scale), and a background worker periodically flushes the current value into the durable Post Store. Reads during the lag window serve the fresher Redis value, not the slightly-stale DB column.

**Terms:**

| Term | Meaning |
|---|---|
| Hot row | A single DB row receiving disproportionate concurrent writes — the bottleneck this design avoids. |
| Hot counter (Redis) | In-memory atomic counter absorbing the actual increment traffic. |
| Write-behind flush | Periodic background job copying the Redis value into the durable store — not synchronous with the like itself. |
| Idempotency key | `user_id + post_id`, checked before incrementing, so a retried/duplicated like request doesn't double-count. |

**Worked timeline** (viral post, flush every 5s):

| Time | Event | Redis counter | Post Store column |
|---|---|---|---|
| t=0 | Post Store column at last known value | — | 1,000 |
| t=0–5s | 500 likes arrive, each does `INCR` in Redis (idempotency-checked) | 1,500 | 1,000 (stale) |
| t=5s | Flush worker copies Redis's value into the Post Store | 1,500 | 1,500 |
| t=5–10s | A page load in this window reads Redis directly (1,500), not the Post Store | 1,500 | 1,500 |

**Correctness pitfall:** the idempotency check (has `user_id` already liked `post_id`?) must happen *before* the `INCR`, and both must be effectively atomic together — otherwise a client retry on a flaky connection (or a double-tap) increments twice for the same user. This is the same class of bug as two separate Redis round-trips in `RateLimiter.md` §5: check-then-act across two calls is a race condition; it needs one atomic operation (e.g. a Lua script doing a conditional `SADD` for the idempotency set plus the `INCR` together).

## 6. Data Model

- Post Store: `post_id, author_id, content_ref, created_at, like_count (periodically flushed)`.
- Hot counter store (Redis): `post:{id}:likes -> int`, `post:{id}:liked_by -> set(user_id)` (idempotency set).
- Follow graph: `follower_id -> [followee_ids]`, `followee_id -> [follower_ids]` (same shape as `NewsFeedSystem.md` §6).

## 7. Scaling, Failure & Trade-offs

- If the flush worker falls behind or dies, Redis remains the correct live value — reads just keep serving from Redis until a worker resumes flushing; no data is lost, only the durable column's freshness is delayed.
- The idempotency set (`liked_by`) grows with the post's total likes; for extremely old/popular posts it can be trimmed or moved to a cheaper store once the post is old enough that new likes are rare.
- This same hot-counter pattern generalizes to comment counts, view counts, and share counts — one Redis key per (post, metric) pair, same flush cadence.

## 8. Wrap-up

Delivered: a write-behind counter design that keeps viral-post engagement off the hot-row path, with an explicit idempotency requirement to prevent double-counting. Not covered due to time: comment threading, abuse detection, and feed ranking (see `NewsFeedSystem.md`).