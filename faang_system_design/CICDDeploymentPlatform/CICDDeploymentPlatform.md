# CI/CD Deployment Platform — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `CICDDeploymentPlatform_component.excalidraw`, `CICDDeploymentPlatform_sequence.excalidraw`.*

## 1. Problem & Scope

Design a platform that builds and deploys code from thousands of repositories, running arbitrary, potentially untrusted pipeline scripts at massive concurrency, without one tenant's build ever contaminating another's.

**In scope:** build isolation, the scheduling/bin-packing of builds onto a worker fleet.

**Explicitly deferred:** the pipeline definition language/DSL, artifact registry internals, deployment rollout strategy (blue/green, canary) beyond the rollback-time NFR.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~10,000 repositories running ~200,000 pipeline executions/day; peak concurrency ~5,000 simultaneous builds.
- **Latency:** median build queue time < 30 s (p99 < 5 min).
- **Recovery:** rollback of a bad release completes within 5 minutes of detection.

**Capacity check:** a pipeline runs arbitrary, caller-defined build scripts — functionally the same trust model as running unknown code, for every single build. At 5,000 simultaneous builds, the scheduling problem (bin-packing builds onto a worker fleet) and the isolation problem (one tenant's build can't observe another's state or secrets) are both real constraints, and conflating "fast build startup" with "safe build startup" is where this design goes wrong if done carelessly.

## 3. Interface

```
POST /pipelines/{repo}/trigger {commit_sha} -> {run_id}
GET /runs/{run_id} -> {status, logs_url}
```

## 4. High-Level Architecture — `CICDDeploymentPlatform_component.excalidraw`

- **Trigger (push/PR) → Build Queue**: incoming build requests wait here for available capacity.
- **Scheduler → Warm Pool**: assigns queued builds to pre-initialized environments when available, for near-instant start.
- **Scheduler → Fresh Environment Provisioner**: spins up a brand-new, isolated environment when the warm pool is exhausted (§5).
- **Build Environment → Artifact Store**: publishes build outputs on success.
- **Deployment Controller → Target Environment**: rolls out (and, on detection of a bad release, rolls back) the artifact.

## 5. Deep Dive — Ephemeral, Isolated Build Environments — `CICDDeploymentPlatform_sequence.excalidraw`

**Options considered:**
- Run every pipeline's build steps directly on shared, long-lived worker machines — fast to start (no environment setup/teardown), but one build's leftover state (cached files, environment variables, installed packages) can leak into or interfere with another tenant's build — both a correctness risk (flaky results) and a security risk (one tenant's code running in a context that could observe another's secrets).
- **Ephemeral, isolated execution environment per build (fresh container/microVM), torn down immediately after — the pick:** guarantees no cross-build contamination or secret leakage; the scheduler's job becomes bin-packing many short-lived environment requests onto available capacity, not managing long-lived worker state.

**Terms:**

| Term | Meaning |
|---|---|
| Ephemeral build environment | A fresh, isolated container/VM created for exactly one build and destroyed immediately after — nothing persists across builds. |
| Build queue | Where pending builds wait for available capacity, ordered to keep p99 queue time bounded. |
| Bin-packing / scheduler | Assigns queued builds to available worker capacity, maximizing utilization without starving any one repository's builds. |
| Warm pool | A small number of pre-initialized-but-unassigned environments kept ready, avoiding full cold-start cost on every build under steady load. |

**Worked timeline** (5,000 builds arrive concurrently):

| Time | Event | Outcome |
|---|---|---|
| t=0 | First wave assigned to warm-pool environments | near-instant start |
| t=0 | Scheduler simultaneously provisions fresh cold environments | replenishes the warm pool and absorbs the excess beyond it |
| t=0+ | Builds exceeding available capacity wait in the queue | per-repository caps on queued/running builds keep one very active repo from starving the other 9,999 |

**Correctness pitfall:** reusing a "cleaned" container between builds instead of a genuinely fresh one (to save cold-start cost) is a common shortcut that reintroduces exactly the cross-build contamination risk isolation was meant to prevent — clearing known paths/env vars is never a complete guarantee against leftover state or secrets, unlike a provably fresh environment instance. The ephemeral guarantee only holds if the environment is actually new, not merely reset.

## 6. Data Model

- Build run: `run_id, repo, commit_sha, status, environment_id, queued_at, started_at`.
- Environment: `environment_id, state (warm|assigned|destroyed), worker_node`.
- Per-repository quota: `repo -> max_concurrent_builds`, enforced by the scheduler to bound fairness.

## 7. Scaling, Failure & Trade-offs

- A worker node failing mid-build just fails that one build (cleanly reported, re-triggerable) — since environments are ephemeral and isolated, no other build sharing that node is affected, unlike a shared long-lived worker going down.
- The warm pool size trades idle compute cost against startup latency — sized to cover typical burst patterns, not peak-of-peak, with cold provisioning absorbing the tail.
- Rollback within 5 minutes requires the Deployment Controller to keep the previous artifact readily deployable (not just archived) — treating rollback as a first-class, pre-tested path, not an emergency improvisation.

## 8. Wrap-up

Delivered: an ephemeral, isolated-per-build execution model with a warm-pool-plus-cold-provisioning scheduler that bounds queue time fairly across repositories. Not covered due to time: the pipeline DSL, artifact registry internals, and deployment rollout strategy details.