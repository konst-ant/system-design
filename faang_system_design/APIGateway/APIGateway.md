# API Gateway — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `APIGateway_component.excalidraw`, `APIGateway_sequence.excalidraw`.*

## 1. Problem & Scope

Design a single entry point that routes client requests to the right backend service, while handling cross-cutting concerns (authN/authZ, rate limiting, validation, versioning) once instead of in every backend.

**In scope:** routing/load balancing, auth delegation, resilience to backend failures (timeouts, retries, cascading-failure protection), horizontal scalability of the gateway tier itself.

**Explicitly deferred:** the rate-limiter's internal algorithm (own exercise — see `RateLimiter.md`), a full API-versioning/deprecation policy, request/response transformation DSLs. These don't change the gateway's core shape.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** gateway fronts ~200 backend services; ~500K RPS at peak platform-wide.
- **Latency:** added latency budget p99 < 10 ms per hop (the gateway must be nearly invisible on the critical path).
- **Availability:** 99.99% — the gateway must never itself become the single point of failure it's meant to protect against.
- **Consistency:** none required at the gateway layer itself — it's a stateless router; the only state worth discussing is per-backend health tracking (§5).

**Capacity check:** 500K RPS of simple routing/header-checking work is CPU-bound, not I/O-bound — a stateless gateway instance can typically handle low tens of thousands of RPS, so ~20–30 instances behind a load balancer cover peak with headroom; scaling is just adding more identical instances, since no shared state is required for routing itself.

## 3. Interface

The gateway doesn't define its own business API — it forwards whatever the backend defines, after stripping/adding standard headers:

```
ANY /{service}/{path} -> routed to backend service "{service}", with auth context injected
```

## 4. High-Level Architecture — `APIGateway_component.excalidraw`

- **Client → Load Balancer → Gateway instance** (any of N identical, stateless instances).
- **Gateway → Auth Service**: validate token/session (or verify a signed JWT locally, no round-trip, when possible).
- **Gateway → Service Registry**: resolve `{service}` name to a live set of backend instances.
- **Gateway → Rate Limiter module**: admission check before forwarding (mechanics in `RateLimiter.md`).
- **Gateway → Backend Service**: the actual forwarded call, protected by the circuit breaker in §5.
- **Gateway → Config Service**: routing rules, version mappings — pulled/cached periodically, not looked up per request.

## 5. Deep Dive — Backend Failure Isolation (Circuit Breaker) — `APIGateway_sequence.excalidraw`

**Options considered:**
- Plain timeout + fixed retries on every call — simple, but under a real backend outage, every gateway instance keeps retrying against it, multiplying load on an already-struggling service ("retry storm") and wasting the gateway's own latency budget waiting on calls that are very likely to fail.
- **Circuit Breaker — the pick:** track each backend's recent failure rate; once it crosses a threshold, stop calling it entirely for a cooldown period and fail fast instead — protects both the backend (less load while it recovers) and the gateway's own latency (no more waiting on doomed calls).

**Terms:**

| Term | Meaning |
|---|---|
| Closed | Normal state — calls pass through to the backend; failures are counted. |
| Open | Tripped state — calls are rejected immediately (fail fast), no network call made to the backend. |
| Failure threshold | Failure rate that trips Closed → Open (e.g. 50% of calls failing in a rolling window). |
| Cooldown | How long the breaker stays Open before testing recovery. |
| Half-Open | Trial state after cooldown — a small sample of calls is let through to probe backend health. |
| Fallback | What the gateway returns while Open (e.g. cached response, a clear 503, or a degraded default) instead of the real backend response. |

**Worked timeline** (threshold = 50% failures, cooldown = 30s):

| Time | Event | Breaker state |
|---|---|---|
| t=0–10s | Backend healthy, calls succeed | Closed |
| t=10–15s | Backend starts failing; failure rate crosses 50% | Closed → **Open** |
| t=15–45s | All calls rejected immediately with fallback, zero load sent to backend | Open |
| t=45s | Cooldown elapses; one sample request let through | **Half-Open** |
| t=45s | Sample request succeeds (backend recovered) | Half-Open → **Closed** |

**Correctness pitfall:** if *every* gateway instance independently probes the backend the instant cooldown ends, that's a thundering herd hitting a backend that just barely recovered, potentially tripping it right back to failing. Fix: only a small, sampled fraction of requests (or a single designated probe) are allowed through during Half-Open — not the full traffic volume — and the sample should be jittered across instances rather than synchronized on the same clock tick.

## 6. Data Model

- Per-backend breaker state (in-memory per gateway instance, not shared — see §7): `state, failure_count, window_start, last_probe_time`.
- Service registry entry: `service_name -> [instance endpoints, health status]`.
- Routing/policy config: `path_prefix -> service_name, version, auth_requirement`.

## 7. Scaling, Failure & Trade-offs

- Breaker state is kept **per gateway instance, not centralized** — each instance makes its own trip/recovery decision independently. This trades perfect synchronization (all instances trip at exactly the same moment) for simplicity and zero extra coordination latency; in practice, with enough traffic per instance, they converge within a few seconds of each other anyway.
- A gateway instance crashing simply drops out of the load balancer's pool — stateless, so no failover coordination needed.
- Auth tokens verified locally (signature check) where possible, to avoid making the Auth Service a dependency on every single request.
- Config/policy changes are cached locally with periodic refresh, same pattern as `RateLimiter.md` §4 — never a synchronous lookup per request.

## 8. Wrap-up

Delivered: a stateless, horizontally-scalable gateway with an explicit circuit-breaker design protecting both backends and the gateway's own latency budget during partial outages. Not covered due to time: full request/response transformation, canary/version routing policy details, and multi-region gateway deployment.
