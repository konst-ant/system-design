# URL Shortener — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `URLShortener_component.excalidraw`, `URLShortener_sequence.excalidraw`.*

## 1. Problem & Scope

Design a service that converts long URLs into short, unique codes and redirects users to the original destination.

**In scope:** short-code generation (no collisions, no coordination overhead), the redirect hot path, basic expiration/custom aliases.

**Explicitly deferred:** click analytics dashboards, spam/malicious-URL detection, custom-domain support. None of these change the core generation/redirect design.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~100M new short URLs created per month (~40 creates/sec average).
- **Read:write ratio:** ~100:1 → ~4K redirects/sec average, ~20K/sec peak.
- **Storage:** over 5 years, ~5B stored mappings at ~500 bytes/record ≈ 2.5 TB.
- **Latency:** redirect path p99 < 100 ms. Creation path can be slower (it's not on anyone's critical path).
- **Availability:** 99.99% for redirects — the redirect path must stay up even if creation is degraded.

**Capacity check:** 20K redirects/sec peak against a key-value lookup (short_code → long_url) is a trivial cache/DB workload — a small Redis layer in front of the mapping store absorbs nearly all of it, given the read-heavy, cacheable access pattern.

## 3. Interface

```
POST /urls {long_url, custom_alias?, expires_at?} -> {short_code}
GET  /{short_code} -> 302 redirect to long_url
```

## 4. High-Level Architecture — `URLShortener_component.excalidraw`

- **Client → API Service**: `POST /urls` → generates a short code (§5) → writes `{short_code, long_url}` to the Mapping Store.
- **Client → Redirect Service**: `GET /{short_code}` → check Cache first, fall back to Mapping Store on miss, populate cache, return 302.
- **API Service → ID Allocator**: checks out a block of IDs (§5) — not a per-request dependency once a block is held locally.
- **Mapping Store**: durable key-value store, partitioned by `short_code`.

## 5. Deep Dive — ID Generation Strategy — `URLShortener_sequence.excalidraw`

**Options considered:**
- Random string + collision check on insert — simple, but as the keyspace fills up, collision probability rises and every create pays for a uniqueness check (and possibly a retry) against the Mapping Store.
- Hash of the long URL (e.g. first 7 chars of MD5) — deterministic, but two different users shortening the same URL collide by design (fine) while hash collisions between *different* URLs still need detection and a fallback, so it doesn't actually remove the collision-handling problem.
- **Range-allocated counter + Base62 encoding — the pick:** a central counter hands out *blocks* of IDs to app servers; each server encodes its own IDs locally. No two servers are ever handed overlapping ranges, so collisions are structurally impossible — no uniqueness check needed on the hot path at all.

**Terms:**

| Term | Meaning |
|---|---|
| Counter service | Single authority handing out non-overlapping ranges of integer IDs. |
| ID block (lease) | A batch of IDs (e.g. 1,000) checked out by one app server at once. |
| Base62 encoding | Maps an integer ID to a short alphanumeric string (`[0-9a-zA-Z]`) — purely a representation, not part of the uniqueness guarantee. |
| Block exhaustion | When a server has used every ID in its current block and must request a new one. |

**Worked timeline:**

| Time | Event | Counter service | Server A's local range | Server B's local range |
|---|---|---|---|---|
| t=0 | Server A requests a block | hands out [1–1000], advances to 1001 | 1–1000 (next: 1) | — |
| t=0 | Server B requests a block concurrently | hands out [1001–2000], advances to 2001 | 1–1000 | 1001–2000 (next: 1001) |
| t=0–5s | A serves 3 creates using IDs 1, 2, 3 locally — no call to the counter service | 2001 (unchanged) | next: 4 | 1001–2000 |
| t=5s | A's block exhausted after ID 1000 → requests a new block | hands out [2001–3000], advances to 3001 | 2001–3000 | 1001–2000 |

Each ID is Base62-encoded into the short code returned to the user (e.g. `1 → "1"`, `125 → "21"`); the encoding is applied locally by whichever server owns that ID, with zero coordination.

**Correctness pitfall:** if Server A crashes after reserving `[2001–3000]` but only uses a few, those IDs are permanently skipped — never reused, never handed to another server. This is the correct behavior, not a bug to fix: trying to reclaim an "abandoned" range would require coordinating with a server that might not actually be dead (just slow), reintroducing exactly the collision risk this design exists to avoid. A sparse ID space is a fine trade for guaranteed uniqueness with zero coordination.

## 6. Data Model

- Mapping Store: `short_code (PK) -> long_url, created_at, expires_at, custom_alias_flag`.
- Counter service: a single row/key holding the next unallocated ID (or a small set of pre-split counters if it ever becomes a bottleneck — unlikely at 40 creates/sec).

## 7. Scaling, Failure & Trade-offs

- Redirect path is read-heavy and highly cacheable (popular links get requested repeatedly) — a cache-aside layer in front of the Mapping Store absorbs most traffic, keeping the store itself lightly loaded.
- Counter service availability matters only for *creates*; if it's briefly down, app servers simply keep serving from whatever block they already hold — creation only stalls once every server's local block is exhausted.
- Custom aliases bypass the counter entirely (user-supplied string) and just need a uniqueness check against the Mapping Store at creation time — a narrow exception to the lock-free path, acceptable since custom aliases are a small fraction of total creates.

## 8. Wrap-up

Delivered: a collision-free ID generation scheme with zero coordination on the hot path, and a cache-backed redirect path meeting the latency/availability targets. Not covered due to time: analytics, abuse detection, and custom-domain routing.