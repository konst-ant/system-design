# Message Storage System at Scale — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `MessageStorageSystem_component.excalidraw`, `MessageStorageSystem_sequence.excalidraw`.*

## 1. Problem & Scope

Design the durable storage and historical retrieval layer for a messaging application — storing and paginating through a conversation's message history at scale.

**In scope:** conversation partitioning, message ordering, cursor-based pagination, hot-conversation handling.

**Explicitly deferred:** live delivery/connection routing to online recipients (own exercise — see `RealTimeChatSystem.md`), message edit/delete semantics, media attachment storage. This system is the *persistence* layer; live delivery is a separate concern.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~500M users; ~20B messages/day (~40/user/day including group chats).
- **Latency:** write-path p99 < 100 ms; history-page read p99 < 200 ms.
- **Durability:** no acknowledged-message loss; multi-year retention for some message classes.
- **Consistency:** strict ordering *within* a conversation; predictable performance even as a single conversation's history grows into the hundreds of thousands of messages.

**Capacity check:** 20B messages/day at an average 200 bytes/message (text + metadata) is ~4 TB/day, ~1.5 PB/year before compression — large but routine for a sharded, append-mostly storage system, as long as each conversation's reads stay localized to its own partition rather than scanning globally.

## 3. Interface

```
POST /conversations/{id}/messages {content} -> {message_id, seq}
GET  /conversations/{id}/messages?before_seq=<seq>&limit=50 -> {messages[], next_cursor}
```

## 4. High-Level Architecture — `MessageStorageSystem_component.excalidraw`

- **Client → Message Service**: send a message or page through history.
- **Message Service → Sequence Allocator**: for a new message, obtains the next per-conversation sequence number (§5) before writing.
- **Message Service → Conversation Store**: durable, partitioned by `conversation_id` — reads and writes always scoped to one conversation's partition.
- **Message Service → Delivery Notifier**: on a durable write, signals the live-delivery layer (`RealTimeChatSystem.md`) that a new message exists — this system's responsibility ends at "durably stored and notified," not delivery itself.

## 5. Deep Dive — Conversation Partitioning & Cursor Pagination — `MessageStorageSystem_sequence.excalidraw`

**Options considered:**
- Single global messages table/index, partitioned only by message ID or write time — simple to write to, but reading "history of conversation C" means filtering across a huge, non-local index; terrible read locality for the dominant access pattern (one conversation's timeline).
- **Partition by `conversation_id`, order by a per-conversation monotonic sequence number — the pick:** every message in one conversation is co-located in the same partition, and a server-assigned sequence number (never wall-clock time) gives strict, gap-free ordering, making "give me the next 50 before sequence N" a cheap, local range scan.

**Terms:**

| Term | Meaning |
|---|---|
| Conversation shard | Partition keyed by `conversation_id` — keeps one conversation's full history co-located. |
| Per-conversation sequence number | Server-assigned, monotonically increasing counter scoped to one conversation — not wall-clock time. |
| Cursor-based pagination | Client passes the last-seen sequence number; server returns the next page strictly before/after it. |
| Hot conversation | A conversation (e.g. a huge group) receiving disproportionate write volume relative to others on the same shard. |

**Worked timeline** (conversation C, pagination page size 50):

| Time | Event | Result |
|---|---|---|
| t=0 | Client requests first page, no cursor | Returns seq 950–999 (latest 50), `next_cursor = 950` |
| t=1 | Client scrolls, requests page `before_seq=950` | Returns seq 900–949, `next_cursor = 900` |
| t=1.5 | A new message arrives concurrently, assigned seq 1001 | Neither already-fetched page shifts — both are anchored to explicit sequence numbers, not a row offset |

**Correctness pitfall:** assigning message order via wall-clock timestamp (especially a client-supplied one) instead of a server-assigned per-conversation sequence is unsafe — two messages sent near-simultaneously from different devices with skewed clocks could be stored out of true order, or even collide on an identical timestamp, making "next page" ambiguous. (Same lesson as `WebCrawler.md` §5 and `FileStorageSyncService.md` §5: anything used for ordering must come from one authoritative, server-side source — never trusted or derived from client state.) The sequence number must be assigned centrally, per conversation, monotonically increasing.

## 6. Data Model

- Message: `conversation_id, seq, message_id, sender_id, content_ref, created_at` — partitioned and physically ordered by `(conversation_id, seq)`.
- Sequence allocator: `conversation_id -> next_seq`, incremented atomically on each new message to that conversation.
- Conversation metadata: `conversation_id, participant_ids, last_seq` (cached for quick "what's the latest" checks without a full read).

## 7. Scaling, Failure & Trade-offs

- A disproportionately active conversation (a huge group) can get its own dedicated partition/throughput allocation rather than sharing a shard sized for typical conversations — the same hot-partition handling already established in `WebCrawler.md` §7 for disproportionately large domains.
- Write durability comes from replicating each partition (same leader/ISR-style replication as `DistributedMessageQueue.md` §5) before acknowledging the write to the client.
- Multi-year retention for some message classes is handled by tiering: recent history stays on fast storage, older partitions migrate to cheaper cold storage, transparently to the pagination API (the cursor doesn't change shape, just the underlying storage tier it reads from).

## 8. Wrap-up

Delivered: a conversation-partitioned store with server-assigned sequence ordering that makes cursor-based pagination cheap and correct at any conversation size. Not covered due to time: live delivery routing (`RealTimeChatSystem.md`), edit/delete semantics, and media storage.