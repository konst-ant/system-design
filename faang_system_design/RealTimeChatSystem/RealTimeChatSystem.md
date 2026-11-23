# Real-Time Chat System — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `RealTimeChatSystem_component.excalidraw`, `RealTimeChatSystem_sequence.excalidraw`.*

## 1. Problem & Scope

Design the *live delivery* layer for a chat application: routing a message to a recipient's active connection the instant it's sent, across a fleet of thousands of stateful connection-gateway instances.

**In scope:** connection registry/presence, live message routing, online/offline delivery branching.

**Explicitly deferred:** durable storage, conversation history, and pagination (own exercise — see `MessageStorageSystem.md`, which this system calls into for every message). This system's job ends at "delivered live, or handed off to offline delivery" — persistence is a separate concern.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~500M DAU; ~100M peak concurrent persistent connections; ~30B messages/day.
- **Latency:** delivery p99 < 200 ms for an online recipient.
- **Delivery guarantee:** eventual delivery for offline recipients — never lost, just delayed.
- **Resilience:** the connection layer must absorb reconnection storms (e.g. after a regional network blip) without cascading failure.

**Capacity check:** 100M concurrent connections spread across thousands of gateway instances means, on average, tens of thousands of connections per instance — routine for a modern event-driven connection server; the actual hard problem is knowing *which* instance holds a given user's connection when another user needs to send them a message.

## 3. Interface

```
WS /connect -> persistent connection, authenticates, registers presence
SEND {conversation_id, content} -> delivered live (if recipient online) or queued for offline delivery
```

## 4. High-Level Architecture — `RealTimeChatSystem_component.excalidraw`

- **Device → Gateway Instance**: holds the live WebSocket connection; registers presence on connect.
- **Gateway Instance → Connection Registry**: `user_id -> {gateway_instance, connection_id}`, shared across all gateways (§5).
- **Sender's Gateway → Connection Registry**: looks up the recipient's current gateway before routing.
- **Sender's Gateway → Recipient's Gateway**: direct internal delivery, only to the one instance that holds the connection.
- **Message Service** (`MessageStorageSystem.md`): every message is durably persisted regardless of whether the recipient is online.

## 5. Deep Dive — Connection Registry & Direct Routing — `RealTimeChatSystem_sequence.excalidraw`

**Options considered:**
- Broadcast every message to every gateway instance, let whichever instance holds the recipient's connection pick it up — simple, but multiplies message volume by the number of gateway instances (thousands), wasting bandwidth at a scale that doesn't survive contact with 30B messages/day.
- **Shared connection registry + direct routing — the pick:** when a user connects, their gateway registers `user_id -> gateway_instance` in a shared store; to deliver a message, the sender's gateway looks up exactly which instance (if any) holds the recipient's live connection and routes there directly — no broadcast.

**Terms:**

| Term | Meaning |
|---|---|
| Connection registry | Shared store mapping a user to which gateway instance currently holds their live connection. |
| Gateway instance | One of many stateful servers holding actual WebSocket connections. |
| Direct routing | Sending a message to exactly the one gateway instance that holds the recipient's connection — never a broadcast. |
| Presence TTL | Registry entries expire on a heartbeat if a gateway crashes without explicitly deregistering, so stale entries don't route messages to a dead connection forever. |

**Worked timeline:**

| Time | Event | Registry state | Outcome |
|---|---|---|---|
| t=0 | User B connects to Gateway 7 | `user_B -> gateway_7` | registered |
| t=1 | User A sends B a message | lookup returns `gateway_7` | routed directly to Gateway 7, delivered to B's socket, p99 < 200 ms |
| t=2 | Gateway 7 crashes | entry not explicitly cleared | — |
| t=2+TTL | Heartbeat-backed entry expires | `user_B` has no registry entry | next message to B finds "no connection" → falls back to offline delivery (push notification, held for B to fetch on reconnect) |

**Correctness pitfall:** if the registry entry for a crashed gateway doesn't expire (no TTL/heartbeat), future messages keep getting routed to a dead instance and silently vanish instead of falling back to offline delivery. This is structurally the same "shared state must be kept honest, not left stale" lesson as `WebCrawler.md` §5's domain timer — just applied to presence instead of rate-limiting.

## 6. Data Model

- Connection registry: `user_id -> {gateway_instance_id, connection_id, last_heartbeat}` — TTL-backed, shared across gateways (e.g. Redis, same role as the shared state stores in `WebCrawler.md`/`RateLimiter.md`).
- Offline delivery queue: `user_id -> [pending message refs]`, drained on reconnect.
- Every message, online or offline path, is written through to `MessageStorageSystem.md`'s conversation store — this system never holds the message as its source of truth.

## 7. Scaling, Failure & Trade-offs

- A reconnection storm (e.g. after a regional blip) means many devices re-register at once; registering is a cheap, independent write per user, so this scales horizontally with gateway count rather than needing special-cased storm handling.
- Typing indicators and read receipts ride the same direct-routing path as messages, but are explicitly *not* durably persisted — losing one during a gateway failover is an accepted, invisible-to-the-user trade-off, unlike an actual message.
- Cross-device delivery (a user on two devices) just means two registry entries for the same user, both targeted on send — no special-casing beyond "a user can have more than one active connection."

## 8. Wrap-up

Delivered: a shared-registry, direct-routing design that avoids broadcast fan-out and degrades cleanly to offline delivery when a gateway fails. Not covered due to time: message persistence/pagination (`MessageStorageSystem.md`), group chat fan-out specifics, and end-to-end encryption key management.