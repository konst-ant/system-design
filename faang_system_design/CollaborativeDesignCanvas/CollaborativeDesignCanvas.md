# Real-Time Collaborative Design Canvas — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `CollaborativeDesignCanvas_component.excalidraw`, `CollaborativeDesignCanvas_sequence.excalidraw`.*

## 1. Problem & Scope

Design a shared visual canvas where many users simultaneously manipulate objects, guaranteeing every client converges to the identical final state with no edit ever silently lost — even after a network partition.

**In scope:** the object data model, concurrent-edit conflict resolution, convergence after reconnection.

**Explicitly deferred:** rendering/canvas performance optimizations, comments/presence cursors beyond basic awareness, version history UI. These layer on top of the conflict-resolution core without changing it.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** boards up to ~5,000 objects; up to 50 concurrent editors per board; ~5K operations/sec platform-wide at peak.
- **Latency:** update-propagation p99 < 100 ms.
- **Convergence:** all clients must reach the identical final state after a network partition heals, with no lost edits.

**Capacity check:** 5K ops/sec platform-wide across many boards is a modest per-board rate even at 50 concurrent editors on one board — the hard part isn't throughput, it's guaranteeing every client ends up agreeing on the final state without a central server having to arbitrate every single conflict in real time.

## 3. Interface

```
WS /boards/{id}/edit {object_id, field, value, logical_clock} -> broadcast to other connected clients
GET /boards/{id} -> {objects: [...], state}
```

## 4. High-Level Architecture — `CollaborativeDesignCanvas_component.excalidraw`

- **Client → Board Session**: each client maintains a local copy of the board and applies/merges operations locally before broadcasting.
- **Board Session → Other Clients**: operations broadcast to every other connected client on the same board.
- **Board Session → Persistence Store**: periodically checkpoints the merged state for durability and reconnecting clients.
- **Reconnecting Client → Board Session**: fetches the latest checkpoint plus any operations it missed, then merges locally.

## 5. Deep Dive — CRDT-Based Conflict Resolution — `CollaborativeDesignCanvas_sequence.excalidraw`

**Options considered:**
- Last-write-wins per object field, using wall-clock time — simple, but two users simultaneously moving/resizing the same object lose one edit entirely, and it's unsafe on its own terms since client clocks can be skewed (same rejected approach as `FileStorageSyncService.md` §5, but more visually jarring here since it's a live drag, not a background sync).
- **CRDT-based object model, per-field LWW register keyed by a logical clock — the pick:** concurrent edits to *different* fields (or different objects) merge automatically with no conflict at all; concurrent edits to the *same* field use a deterministic, commutative merge rule — whichever operation has the higher logical clock wins, the same result on every client regardless of delivery order. No central coordinator needs to arbitrate.

**Terms:**

| Term | Meaning |
|---|---|
| CRDT | Conflict-free Replicated Data Type — a structure whose merge operation is commutative and idempotent, so any delivery order converges to the same result. |
| Logical clock | A per-client, per-operation counter used to break ties deterministically — never wall-clock time. |
| Per-field LWW register | Each object property (e.g. x-position) merges independently using logical-clock-based last-write-wins. |
| Convergence | The guarantee that all clients reach the identical final state once they've seen the same set of operations, regardless of order. |

**Worked timeline** (object O starts at `x=100, y=100`):

| Clock | Event | Result on every client |
|---|---|---|
| tick 5 (Client A) | Sets `x=150` | — |
| tick 4 (Client B) | Sets `y=200` (different field) | No conflict: `O = {x=150, y=200}` once both ops are seen, regardless of order |
| tick 5 (Client A) vs tick 6 (Client B) | Both set `x` concurrently — A: `x=150`, B: `x=175` | B's tick (6) is higher → **`x=175` wins on every client**, no matter which operation each client received first |

**Correctness pitfall:** using wall-clock timestamps instead of a logical clock for the per-field tie-break reintroduces exactly the clock-skew bug already flagged in `RateLimiter.md` §5 and `FileStorageSyncService.md` §5 — a client with a merely faster wall clock could "win" over an edit that actually happened later in real causal order, producing merge results that don't match what either user actually did.

## 6. Data Model

- Object: `object_id, {field: {value, logical_clock}}` — every field independently versioned, not the object as a whole.
- Logical clock: `client_id, counter` — incremented locally on every local edit, included with every broadcast operation.
- Checkpoint: periodic durable snapshot of the fully-merged board state, used to bound how far a reconnecting client has to replay.

## 7. Scaling, Failure & Trade-offs

- Because merging is commutative, a client that was disconnected can simply apply every operation it missed, in *any* order, and still converge — no replay-order coordination needed, unlike the deterministic-replay requirement in `WorkflowOrchestrationEngine.md` §5.
- A board session server crash loses no data beyond the last checkpoint interval — clients reconnect, pull the last checkpoint, and resume broadcasting; a short gap in live collaboration, not a correctness issue.
- Objects-per-board and editors-per-board are both bounded (5,000 / 50) in this NFR, which keeps the full-state checkpoint small enough to ship to a reconnecting client in one shot rather than needing incremental/partial sync.

## 8. Wrap-up

Delivered: a CRDT-based, logical-clock-ordered merge model that guarantees convergence without a central arbiter, with an explicit rejection of wall-clock-based tie-breaking. Not covered due to time: rendering performance, presence/cursor broadcasting beyond basic awareness, and version history.