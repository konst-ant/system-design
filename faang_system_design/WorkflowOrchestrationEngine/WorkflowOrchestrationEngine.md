# Workflow Orchestration Engine — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `WorkflowOrchestrationEngine_component.excalidraw`, `WorkflowOrchestrationEngine_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform that executes long-running business workflows (hours to months), where individual activities can fail and retry independently, without ever losing track of where a workflow is or re-applying a business effect twice.

**In scope:** durable workflow state, crash recovery, exactly-once business effects despite at-least-once activity retries.

**Explicitly deferred:** the workflow definition language/DSL, human-in-the-loop approval UI, cross-workflow signaling patterns. These layer on top without changing the durability core.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~100K concurrently active workflow instances, each generating up to a few hundred history events.
- **Duration:** individual workflows may run from minutes to months.
- **Latency:** activity-dispatch latency p99 < 1 s.
- **Durability:** workflow state must survive worker and orchestrator restarts with zero lost progress.

**Capacity check:** 100K workflows × a few hundred events each is on the order of 10M-100M total history events resident — a modest dataset for a durable log store, especially since most workflows are idle between activities (waiting on a timer or an external event) rather than actively computing, so the real load is bursty dispatch traffic, not sustained CPU.

## 3. Interface

```
START /workflows {type, input} -> {workflow_id}
SIGNAL /workflows/{id} {event} -> ack          // external event delivered to a running workflow
GET /workflows/{id} -> {status, history}
```

## 4. High-Level Architecture — `WorkflowOrchestrationEngine_component.excalidraw`

- **Client → Orchestrator API**: starts/signals workflows.
- **Orchestrator → Event History Store**: durable, append-only log per workflow instance (§5).
- **Orchestrator → Task Queue**: dispatches the next activity a workflow needs, picked up by any available Worker.
- **Worker → Activity code**: executes one activity, reports completion/failure back to the Orchestrator.
- **Orchestrator (on recovery)**: replays a workflow's event history through its (deterministic) code to reconstruct in-memory state — never reads a stored "current state" directly.

## 5. Deep Dive — Durable Execution via Event Sourcing — `WorkflowOrchestrationEngine_sequence.excalidraw`

**Options considered:**
- Store only the workflow's "current state" (a row updated in place at each step) — simple, but a crash mid-step leaves ambiguous state: did the activity actually run before the crash, or not? Recovering correctly requires guessing or expensive external checks.
- **Event-sourced history + deterministic replay — the pick:** every decision (activity started, activity completed, timer fired) is durably appended to an append-only log; in-memory workflow state is *never* stored directly — it's reconstructed by replaying the log through the workflow's own (deterministic) code. Crash recovery is just "replay the log."

**Terms:**

| Term | Meaning |
|---|---|
| Event history | Append-only, durable log of everything that has happened to one workflow instance. |
| Deterministic replay | Re-running the workflow's code against its recorded history to reconstruct current state — requires the code to produce identical results every time given the same history. |
| Activity | One unit of external work (may call a real API, write to a DB, etc.) — potentially retried. |
| Activity ID (idempotency key) | Lets the engine recognize "this activity already ran" during a retry or replay, so it isn't re-executed with a duplicate side effect. |

**Worked timeline:**

| Time | Event | Event history | Outcome |
|---|---|---|---|
| t=0 | Workflow dispatches Activity A, appends `ActivityAStarted` | `[...,ActivityAStarted]` | Worker begins executing A |
| t=1 | Worker crashes before reporting completion | `[...,ActivityAStarted]` (unchanged) | Orchestrator notices missed heartbeat |
| t=2 | Orchestrator re-dispatches A with the **same activity ID** to a new worker | unchanged | New worker checks "has activity ID X already applied its effect?" — idempotent, safe to retry |
| t=3 | A completes, appends `ActivityACompleted` | `[...,ActivityAStarted,ActivityACompleted]` | Workflow progresses; a later replay reconstructs this exact same state deterministically |

**Correctness pitfall:** if workflow code is *not* deterministic — e.g. it calls `random()` directly, or reads the system wall clock instead of going through the engine's time API — replaying the same event history on recovery can take a *different* execution path than the original run, silently corrupting the workflow's logic. This is why orchestration frameworks force all non-deterministic operations (time, randomness, UUIDs) through engine-provided equivalents that get recorded *in* the event history itself, rather than re-evaluated fresh on every replay.

## 6. Data Model

- Event history: `workflow_id, event_seq, event_type, payload, timestamp` — append-only, ordered by `event_seq`.
- Activity record: `activity_id, workflow_id, status, result` — the idempotency anchor for retries.
- Task queue entry: `workflow_id, activity_id, dispatch_time` — removed once a worker claims and reports on it.

## 7. Scaling, Failure & Trade-offs

- Workflows are sharded across orchestrator instances by `workflow_id`; an orchestrator crashing just means its workflows get picked up and replayed by another instance — no special-cased recovery logic beyond "replay the log."
- Workers are stateless and horizontally scalable — any worker can pick up any dispatched activity, since all the state that matters lives in the durable event history, not in worker memory.
- Very long-running workflows (months) periodically checkpoint/compact their history (e.g., via a "continue-as-new" pattern) so replay time doesn't grow unboundedly with age.

## 8. Wrap-up

Delivered: an event-sourced, replay-based durability model that survives crashes at any point without ambiguity, with an explicit determinism requirement flagged as the one way to break it. Not covered due to time: the workflow DSL itself, human-approval steps, and cross-workflow signaling patterns.