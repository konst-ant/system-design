# Recommendation System — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `RecommendationSystem_component.excalidraw`, `RecommendationSystem_sequence.excalidraw`.*

## 1. Problem & Scope

Design a system that returns a personalized, ranked list of items for a user, under a strict per-request latency budget, from a catalogue far too large to fully score on every request.

**In scope:** the online serving path (candidate generation + ranking), and the offline/online data split that makes it feasible.

**Explicitly deferred:** the ranking model's internal architecture (any reasonable model plugs into the same funnel), A/B testing of ranking variants (own exercise — see the A/B Testing Platform exercise), content-based cold-start heuristics in detail.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~200M users; catalogue of ~50M items; ~500K recommendation requests/sec at peak.
- **Latency:** online inference p99 < 100 ms, including feature lookups.
- **Freshness:** offline models retrain on an hours-to-daily cycle; online behavioral signals (clicks, skips) should influence results within minutes.

**Capacity check:** scoring all 50M items for every one of 500K requests/sec is computationally impossible within a 100 ms budget — even at a generous 1 µs/item scoring cost, 50M items would take 50 ms *per request* for the cheapest possible score, before any feature lookup or network overhead, and real ranking models cost far more than 1 µs/item. The request volume and catalogue size together rule out brute-force scoring outright.

## 3. Interface

```
GET /recommendations?user_id=<id>&context=<ctx> -> { items: [{item_id, score}], ... }
```

## 4. High-Level Architecture — `RecommendationSystem_component.excalidraw`

- **Client → Recommendation Service**: single entry point for a request.
- **Recommendation Service → Candidate Generator**: cheap, approximate retrieval narrowing the catalogue down (§5).
- **Recommendation Service → Ranking Service**: scores only the narrowed candidate set with richer features.
- **Feature Store**: serves both stages their respective features with low latency.
- **Offline Training Pipeline**: retrains the embedding model (feeds Candidate Generator) and the ranking model on an hours-to-daily cycle, independently of each other.

## 5. Deep Dive — Two-Stage Retrieval Funnel — `RecommendationSystem_sequence.excalidraw`

**Options considered:**
- Score every catalogue item for every request ("brute-force ranking") — would produce the globally optimal order, but is computationally infeasible at this scale (§2) — ruled out immediately, not a real trade-off.
- **Two-stage funnel — the pick:** Stage 1 (candidate generation) cheaply narrows ~50M items down to a few hundred likely-relevant candidates; Stage 2 (ranking) spends its compute budget only on that much smaller set, using richer features.

**Terms:**

| Term | Meaning |
|---|---|
| Candidate generation (retrieval) | Stage 1 — cheap, approximate narrowing of the full catalogue to a few hundred candidates. |
| Embedding | A learned vector representation of a user or item, enabling fast similarity lookups. |
| Approximate nearest neighbor (ANN) index | A data structure that finds "items near this user's embedding" in roughly-sorted order without comparing against every item. |
| Ranking model | Stage 2 — a more expensive model, scoring only the candidate set with full feature richness. |
| Funnel | The overall pattern of narrowing counts at each stage: 50M → few hundred → final top-20. |

**Worked timeline** (single request, 100 ms budget):

| Stage | Work done | Items considered | Time spent |
|---|---|---|---|
| Stage 1 | ANN lookup against the user's embedding | 50M → 500 candidates | ~10 ms |
| Stage 2 | Ranking model scores the 500 candidates with full features | 500 → top 20 | ~40 ms |
| — | Feature lookups + response assembly | — | ~20 ms |
| **Total** | | | **~70 ms**, within the 100 ms budget |

Compare this to brute-force scoring of 50M items, which the §2 capacity check already ruled out as physically impossible in this budget — the funnel isn't an optimization of that approach, it's a structurally different one.

**Correctness pitfall:** the two stages are refreshed independently — if a brand-new item is never added to the Stage 1 embedding index (e.g. an indexing job bug, or a delay in the embedding pipeline), that item is permanently unreachable, *no matter how good Stage 2's ranking is*, because it will never appear in anyone's candidate set. This is a distinct failure mode from "cold-start user" (a user with no history) — it's a cold-start *item* problem, and it's easy to miss in review because Stage 2 looks like it's working perfectly on whatever candidates it happens to receive.

## 6. Data Model

- Item embedding index (ANN-searchable): `item_id -> embedding_vector`, rebuilt/updated as part of the offline pipeline.
- User embedding: `user_id -> embedding_vector`, updated on the same or a faster cycle.
- Feature store: `(user_id | item_id) -> feature_vector`, serving both stages with a shared low-latency lookup path.

## 7. Scaling, Failure & Trade-offs

- Candidate generation and ranking scale independently — the ANN index is sharded/replicated for read throughput; the ranking model is a stateless scoring service scaled by request volume.
- If the Ranking Service is briefly degraded, falling back to Stage 1's raw candidate order (unranked but still relevant) is a reasonable degraded mode — better than failing the request outright.
- Online signal freshness (minutes) vs. model retraining freshness (hours-daily) is an explicit, accepted two-speed design: a lightweight online feature (e.g. "did they just skip this?") can adjust ranking immediately, while the underlying model itself only needs to adapt on a slower cycle.

## 8. Wrap-up

Delivered: a two-stage funnel that makes recommendation at this scale computationally feasible, with an explicit, easy-to-miss cold-start-item failure mode called out. Not covered due to time: the ranking model's internals, A/B testing integration, and detailed cold-start-user heuristics.