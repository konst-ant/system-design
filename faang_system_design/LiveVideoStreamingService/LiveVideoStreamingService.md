# Live Video Streaming Service — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `LiveVideoStreamingService_component.excalidraw`, `LiveVideoStreamingService_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform that broadcasts live video to audiences that can scale from a handful of viewers to millions within minutes, without the origin ever seeing load proportional to viewer count.

**In scope:** ingest-to-distribution pipeline, CDN edge fan-out under a sudden viral surge.

**Explicitly deferred:** live chat, transcoding ladder details (same renditions/chunking mechanics as `VideoStreamingPlatform.md` §5, just continuously produced instead of pre-computed), DVR/replay.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** a single viral stream may reach ~5M concurrent viewers.
- **Latency:** glass-to-glass (broadcaster to viewer) 3–10 s depending on low-latency mode.
- **Delivery:** each stream transcoded into at least 3 bitrate renditions.
- **Availability:** 99.9% once live; must absorb a 10x viewer surge within minutes without degrading playback for existing viewers.

**Capacity check:** a live stream's segments are generated exactly once, continuously, and can't be pre-warmed into a CDN ahead of time the way VOD content can (`VideoStreamingPlatform.md` §2) — if even a small fraction of millions of viewers pulled directly from the one origin encoder, it would be overwhelmed instantly regardless of how much origin capacity is provisioned.

## 3. Interface

```
INGEST stream={id} -> origin encoder (broadcaster's feed)
GET /live/{stream_id}/manifest -> {renditions, segment_url_pattern}
GET /segments/{stream_id}/{rendition}/{seq}.ts -> binary chunk (served via CDN hierarchy)
```

## 4. High-Level Architecture — `LiveVideoStreamingService_component.excalidraw`

- **Broadcaster → Origin Encoder**: ingests the raw feed, transcodes into renditions, publishes segments as they're produced.
- **Origin → Regional Shield Caches**: a small, fixed number of mid-tier caches, each pulling each segment from the origin at most once.
- **Shield Cache → Edge PoPs**: many edge points-of-presence pull from the nearest shield, not the origin.
- **Edge PoP → Viewers**: serves actual viewer requests; all same-PoP viewers share one cached copy of each segment.

## 5. Deep Dive — Hierarchical Edge Fan-out & Origin Shielding — `LiveVideoStreamingService_sequence.excalidraw`

**Options considered:**
- Every edge PoP pulls directly from the origin for every viewer request — simple, but a viral surge means thousands of PoPs hitting the one origin simultaneously for the same live segment, which can't be replicated ahead of time the way VOD content can.
- **Hierarchical distribution (origin → regional shields → edge PoPs), each tier caching and sharing — the pick:** origin is shielded by a small number of regional caches; many edge PoPs pull from the nearest shield instead of the origin, and within one PoP, the first viewer's request populates a cache that every other same-PoP viewer then shares.

**Terms:**

| Term | Meaning |
|---|---|
| Origin | The single source producing live segments from the transcoder — never scales with viewer count. |
| Regional shield cache | Mid-tier absorbing nearly all repeat pulls, so the origin sees bounded load regardless of total viewers. |
| Edge PoP | Point of presence nearest viewers; serves the actual viewer requests. |
| Segment | An immutable, few-seconds chunk of the live stream — same unit as VOD's chunk, but continuously produced and cached only briefly. |

**Worked timeline** (viewer count ramps 10K → 5M in minutes):

| Viewers | Event | Origin load |
|---|---|---|
| 10K | First request for segment N at each of several PoPs → each PoP's first miss triggers one shield fetch; shield's first miss triggers one origin fetch | ~1 request per segment per shield |
| 500K | More PoPs now active, but each still only misses to its shield *once* per segment; within a PoP, all viewers share the cached copy | unchanged — still ~1 request per segment per shield |
| 5M | Surge continues; new PoPs spin up and each does its own single shield fetch | still bounded by the fixed number of shields, not viewer count |

**Correctness pitfall:** caching a segment keyed only by `stream_id` (not `stream_id + segment_sequence`) risks serving a stale or wrong segment to new viewers as new segments continuously replace old ones under the same stream. Segments must be immutable once published and cached by their full `(stream_id, segment_sequence)` key — the same immutability principle as `VideoStreamingPlatform.md` §5's GOP-aligned renditions, just with a much shorter useful cache lifetime (seconds, not indefinite).

## 6. Data Model

- Segment: `stream_id, rendition, sequence, data` — immutable once published, short TTL once cached at any tier.
- Manifest: `stream_id -> {renditions, latest_sequence}`, polled or pushed to viewers to discover new segments.
- Shield/edge cache entries: keyed by `(stream_id, rendition, sequence)`, evicted shortly after the segment is no longer the live edge.

## 7. Scaling, Failure & Trade-offs

- A shield cache failing fails over to another shield or, worst case, the origin directly for a brief window — bounded blast radius since there are only a handful of shields, not thousands of PoPs hitting origin at once.
- Lower glass-to-glass latency (closer to 3s) means shorter segments and less time for the caching hierarchy to populate before a viewer wants that segment — a direct latency-vs-cache-efficiency trade-off, which is why ultra-low-latency modes cost more origin/shield capacity per viewer than standard live latency.
- A sudden 10x surge is absorbed because the hierarchy's fan-out ratio at each tier (one shield serving many PoPs, one PoP serving many viewers) is what scales — the origin's load is a function of segment count and shield count, never viewer count.

## 8. Wrap-up

Delivered: a shielded, hierarchical CDN design that keeps origin load constant regardless of viewer count, with an explicit cache-key correctness requirement for continuously-produced segments. Not covered due to time: live chat, detailed transcoding ladder mechanics, and DVR/replay.
