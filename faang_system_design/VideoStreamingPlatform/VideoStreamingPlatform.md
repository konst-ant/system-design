# Video Streaming Platform — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `VideoStreamingPlatform_component.excalidraw`, `VideoStreamingPlatform_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform that lets users upload, browse, and stream pre-recorded video with minimal startup delay and buffering across varying network conditions.

**In scope:** upload/transcode pipeline, storage/CDN delivery, and — as the deep dive — adaptive bitrate playback.

**Explicitly deferred:** recommendations (own exercise — see `RecommendationSystem.md`), live streaming (a different problem — see the Live Video Streaming exercise), content moderation. None change the transcode/delivery core.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~200M DAU; ~1M concurrent streams at peak; catalogue of ~1M titles, each transcoded into 5–10 renditions.
- **Latency:** playback-start p99 < 2 s; rebuffer ratio < 0.5%.
- **Delivery:** CDN cache-hit ratio > 95% for popular content.
- **Durability:** uploaded source video must never be lost; transcoded renditions can always be regenerated from source if needed.

**Capacity check:** 1M concurrent streams at an average bitrate of ~3 Mbps is ~3 Tbps of aggregate egress — this is only feasible because a CDN serves the >95% cache-hit traffic from edge nodes near the viewer; the origin only ever needs to serve cache misses and long-tail content, a small fraction of the total.

## 3. Interface

```
POST /videos {file} -> {video_id}              // triggers async transcode
GET  /videos/{id}/manifest -> {renditions: [{bitrate, url}, ...], segment_duration}
GET  /segments/{video_id}/{rendition}/{n}.ts -> binary chunk  // served via CDN
```

## 4. High-Level Architecture — `VideoStreamingPlatform_component.excalidraw`

- **Uploader → Upload Service → Raw Storage**: original file landed durably before anything else happens.
- **Transcoding Pipeline**: consumes raw uploads, produces multiple bitrate renditions, each split into fixed-length chunks (§5), writes to Rendition Storage.
- **Manifest Service**: on upload completion, publishes a manifest listing available renditions and chunk URLs.
- **CDN**: caches rendition chunks at edge locations close to viewers.
- **Client Player → CDN**: fetches the manifest once, then fetches chunks continuously, choosing rendition per chunk (§5).

## 5. Deep Dive — Adaptive Bitrate Streaming — `VideoStreamingPlatform_sequence.excalidraw`

**Options considered:**
- Fixed single rendition chosen once at playback start, based on an initial bandwidth probe — simple, but doesn't adapt if the network degrades or improves mid-playback, causing stalls on networks that get worse after the probe.
- **Chunked adaptive bitrate (HLS/DASH-style) — the pick:** video is pre-encoded into multiple bitrate renditions, each split into short chunks (e.g. 4–6 s); the client measures its own recent download throughput and buffer health, and picks the rendition for *each upcoming chunk* independently.

**Terms:**

| Term | Meaning |
|---|---|
| Rendition | One full bitrate/resolution encoding of the video (e.g. 1080p, 480p). |
| Chunk (segment) | A short, fixed-duration slice of one rendition — the unit the client actually requests. |
| Manifest | Lists available renditions and the URL pattern for fetching each chunk. |
| Buffer health | How many seconds of already-downloaded-but-unplayed video the client is holding. |
| ABR algorithm | Client-side logic choosing the next chunk's rendition from recent throughput + buffer health. |
| GOP alignment | All renditions share identical keyframe boundaries at every chunk boundary — required for switching mid-stream. |

**Worked timeline** (4 s chunks):

| Chunk | Event | Rendition chosen | Buffer health after |
|---|---|---|---|
| 1 | Fetched in 2 s (bandwidth comfortably covers 1080p) | 1080p | healthy — ahead of real-time |
| 2 | Bandwidth drops; fetch takes 5 s for a 4 s chunk (falling behind) | 1080p (still, decided before the drop was known) | buffer shrinking |
| 3 | Client detects the slow fetch, switches down | 480p | buffer recovers |
| 4 | Bandwidth still low, fetch keeps up at this rate | 480p | stable |

**Correctness pitfall:** rendition switches may only happen on a chunk boundary — never mid-chunk. This is only possible because every rendition is encoded with identical keyframe/GOP boundaries aligned to the same chunk timestamps; if renditions were encoded independently without that alignment, switching mid-stream would produce a visible glitch or audio/video desync. This constraint has to be enforced at encode time in the transcoding pipeline — it cannot be patched after the fact at playback time.

## 6. Data Model

- Video metadata: `video_id, title, upload_status, duration, available_renditions`.
- Manifest: `video_id -> [{rendition, bitrate, chunk_url_pattern, chunk_count}]`.
- Rendition storage: object storage keyed by `video_id/rendition/chunk_index`.

## 7. Scaling, Failure & Trade-offs

- Transcoding is CPU-heavy and fully parallelizable per rendition — a failed transcode job for one rendition is retried independently without blocking the others or delaying publication of renditions that succeeded.
- CDN cache misses fall back to the origin; the origin only needs capacity for the long-tail (~5% of traffic), not the full peak.
- Startup latency is dominated by manifest fetch + first chunk fetch — keeping the first chunk small/low-bitrate specifically (even if later chunks go higher) is a common trick to hit the playback-start latency target without compromising steady-state quality.

## 8. Wrap-up

Delivered: a chunked ABR design with an explicit encode-time constraint (GOP alignment) that makes seamless rendition switching possible. Not covered due to time: recommendations, live streaming, and content moderation.