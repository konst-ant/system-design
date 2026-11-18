# File Storage & Synchronisation Service — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `FileStorageSyncService_component.excalidraw`, `FileStorageSyncService_sequence.excalidraw`.*

## 1. Problem & Scope

Design a cloud storage platform that lets users upload, organize, and synchronize files across multiple devices, correctly handling the case where two devices edit the same file while both are offline.

**In scope:** chunked upload/storage, and — as the deep dive — multi-device sync conflict detection and resolution.

**Explicitly deferred:** sharing/permissions logic, deduplication internals, folder-hierarchy metadata design. None change the conflict-resolution core.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~100M users; ~10 GB stored/user on average (~1 EB total); ~1M sync operations/sec at peak across all clients.
- **Latency:** small-file upload p99 < 1 s; large files use chunked, resumable transfer.
- **Correctness:** a user's edit must never be silently discarded — this is a storage product; data loss is a hard failure, not a tunable trade-off.
- **Convergence:** devices reconnecting after being offline converge to a consistent state within seconds.

**Capacity check:** with millions of devices syncing independently and intermittently, any two devices can plausibly edit the same file while both are offline — this isn't a rare edge case to handle defensively, it's a routine occurrence the core design must handle correctly every time, not just detect after the fact.

## 3. Interface

```
PUT /files/{id} {base_version, chunks} -> { new_version } | { conflict: true, conflicted_copy_id }
GET /files/{id} -> { version, chunks, metadata }
```

## 4. High-Level Architecture — `FileStorageSyncService_component.excalidraw`

- **Device → Sync Client**: watches local changes, chunks modified files, pushes updates.
- **Sync Client → Sync Service**: presents the file's `base_version` (what the edit was derived from) along with the new content.
- **Sync Service → Metadata Store**: holds the authoritative current version per file (§5).
- **Sync Service → Chunk Store**: content-addressed object storage for actual file data (enables deduplication across users/versions).
- **Sync Service → Device B (push notification)**: other devices are notified of the new version to pull.

## 5. Deep Dive — Multi-Device Conflict Detection — `FileStorageSyncService_sequence.excalidraw`

**Options considered:**
- Last-write-wins by timestamp — simple, but silently discards one device's edit whenever two devices modify the same file offline and reconnect with conflicting versions; dangerous for a storage product where data loss is unacceptable. It's also unsafe on its own terms: client clocks can be skewed, so a device with a merely *faster* clock could "win" over a genuinely newer edit.
- **Version vector + conflicted copy on divergence — the pick:** every file has a server-assigned, monotonically increasing version number. A device's sync request must present the version its edit was based on (`base_version`). If the server's current version has moved past that, it's a conflict: the server's existing version proceeds untouched, and the incoming edit is saved as a separate "conflicted copy" — never silently dropped.

**Terms:**

| Term | Meaning |
|---|---|
| Version number | Server-assigned, monotonically increasing counter per file — the only thing conflict detection trusts. |
| Base version | The version a device's edit was derived from — sent along with every sync request. |
| Conflict | Detected when `base_version != server's current version` at sync time. |
| Conflicted copy | A new file created to hold an edit that lost the version race — preserves the data instead of discarding it. |
| Chunk | Content-addressed block of file data, enabling dedup across versions/users. |

**Worked timeline** (file F starts at version 3, both devices last synced at v3):

| Time | Event | Server version of F |
|---|---|---|
| t=0 | Device A (offline since v3) edits F, reconnects, syncs with `base_version=3` | server still at 3 → no conflict → accepted, becomes **v4** |
| t=1 | Device B (also offline since v3) edits F independently, reconnects, syncs with `base_version=3` | server now at 4 → **mismatch → conflict** |
| t=1 | Server's response to B | F stays at v4 (Device A's edit); B's edit is saved as `"F (conflicted copy from Device B)"` — nothing lost |

**Correctness pitfall:** detecting conflicts by comparing *timestamps* instead of server-assigned version numbers is unsafe, because client clocks can be skewed or simply wrong — a device with a fast clock could appear to "win" over an edit that actually happened later. The version number must be assigned and incremented by the server on each accepted sync, never supplied or trusted from the client. (Same lesson as `WebCrawler.md` §5's rejection of client-local timers: anything used for ordering/conflict decisions must come from a single authoritative source, not from individually-trusted client state.)

## 6. Data Model

- File metadata: `file_id, current_version, chunk_list, owner_id, updated_at`.
- Chunk store: content-addressed `chunk_hash -> chunk_data` (dedup is automatic — identical content across files/users maps to the same chunk).
- Conflicted copy: a regular file record, linked to the original `file_id` for UI grouping, but otherwise independent.

## 7. Scaling, Failure & Trade-offs

- Chunking + content-addressing means only the changed chunks of a large file need to be re-uploaded on a small edit, not the whole file — this is what makes the p99 < 1 s target achievable even for files that are themselves much larger.
- A device that was offline for a long time doesn't need special-case recovery — it just presents a stale `base_version` on reconnect, and the normal conflict path handles it identically to a short offline gap.
- Conflicted copies accumulate if users don't manually resolve them; this is an accepted UX trade-off (surface them clearly) rather than a reason to risk silent data loss by picking a different resolution strategy.

## 8. Wrap-up

Delivered: a version-vector-based conflict model that guarantees no edit is ever silently lost, with an explicit rejection of timestamp-based ordering. Not covered due to time: sharing/permissions, deduplication internals beyond content-addressing, and folder hierarchy metadata.