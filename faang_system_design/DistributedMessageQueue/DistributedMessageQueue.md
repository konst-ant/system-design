# Distributed Message Queue — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `DistributedMessageQueue_component.excalidraw`, `DistributedMessageQueue_sequence.excalidraw`.*

## 1. Problem & Scope

Design a messaging platform where producers publish events and consumers process them asynchronously, with a hard guarantee that an acknowledged message is never lost, even across broker failures.

**In scope:** topic/partition model, replication and acknowledgment strategy, consumer group semantics.

**Explicitly deferred:** exactly-once processing semantics end-to-end (own exercise — see `RealTimeDataIngestionPipeline.md` for the consumer-side windowing concerns), schema registry/evolution, cross-datacenter replication.

## 2. NFRs & Back-of-the-Envelope

- **Scale:** ~2M messages/sec aggregate, average message size 1 KB (~2 GB/sec ingest).
- **Retention:** 7 days → ~1.2 PB retained at steady state (`2 GB/s × 604,800s`).
- **Latency:** end-to-end publish-to-consume p99 < 50 ms for latency-sensitive topics.
- **Durability:** zero loss of acknowledged messages across broker failures — this is a hard requirement, not a tunable trade-off.

**Capacity check:** 1.2 PB retained across, say, ~1,000 partitions means ~1.2 TB/partition — comfortably within a single broker's local disk, so retention is a storage-provisioning problem, not an architectural one; the real engineering problem is making the *durability* guarantee hold during the failures that will inevitably happen at this scale.

## 3. Interface

```
PRODUCE topic={t} partition_key={k} payload -> {offset}
CONSUME topic={t} consumer_group={g} -> {messages[], offsets[]}  // commit offset after processing
```

## 4. High-Level Architecture — `DistributedMessageQueue_component.excalidraw`

- **Producer → Leader broker**: every partition has one leader; all writes for that partition go there.
- **Leader → Follower brokers**: replicates each write to the partition's replica set before acknowledging (§5).
- **Consumer Group Coordinator**: tracks which consumer instance owns which partition, rebalances on failure.
- **Consumer → Leader broker**: reads sequentially from its assigned partitions, commits offsets as it progresses.
- **Metadata Store** (e.g. a small Raft/ZooKeeper-style quorum): tracks current leader and ISR membership per partition.

## 5. Deep Dive — Replication & Acknowledgment Strategy — `DistributedMessageQueue_sequence.excalidraw`

**Options considered:**
- No replication, ack on local write only — fast, but a single broker failure loses every unreplicated message instantly; violates the durability NFR outright.
- Synchronous replication to *every* replica before ack — maximizes durability, but one slow or temporarily-down replica blocks every write to the partition, killing availability and latency.
- **Leader + in-sync replica set (ISR) with a configurable ack quorum — the pick:** a write is acknowledged once a quorum of *currently healthy* replicas confirm it, not all replicas ever provisioned — a lagging replica is dropped from the ISR rather than allowed to block every write.

**Terms:**

| Term | Meaning |
|---|---|
| Leader | The single broker a partition's writes and reads go through. |
| ISR (in-sync replica set) | The subset of replicas currently caught up enough to count toward durability — shrinks automatically if a replica falls behind. |
| Ack quorum (`min.insync.replicas`) | Minimum ISR size required before a write is acknowledged to the producer. |
| High-water mark | The offset up to which every ISR member has replicated — the only offsets safe to expose to consumers. |

**Worked timeline** (`min.insync.replicas = 2`, replication factor 3):

| Time | Event | ISR | Outcome |
|---|---|---|---|
| t=0 | Leader + 2 followers healthy, producer sends message | {L, F1, F2} | Ack after 2 of 3 confirm — quorum met |
| t=1 | F2 falls behind (slow disk), dropped from ISR | {L, F1} | Still meets `min.insync.replicas=2` — writes continue uninterrupted |
| t=2 | F1 also falls behind | {L} | ISR size (1) < 2 → **producer gets an error**, not a silent durability downgrade |

**Correctness pitfall:** if an out-of-ISR broker is allowed to become leader anyway during a failover ("unclean leader election," enabled for availability), any message acknowledged to the producer but never replicated to that broker is silently lost the moment it takes over — the ack lied. This must be disabled whenever durability is a hard requirement, accepting the availability cost (no new leader can be elected until an ISR member returns) as the explicit trade-off, not something to quietly relax under pressure.

## 6. Data Model

- Partition log (per partition): append-only sequence of `(offset, key, payload, timestamp)`, immutable once written.
- ISR metadata: `partition -> {leader, isr_set, high_water_mark}`, kept in the Metadata Store, not on the data brokers themselves.
- Consumer group offsets: `(group_id, partition) -> committed_offset`.

## 7. Scaling, Failure & Trade-offs

- Partitioning is the unit of both parallelism and ordering: ordering is only guaranteed *within* a partition, never across partitions — a deliberate trade-off for throughput.
- A consumer crashing simply triggers a group rebalance; since offsets are committed externally (not held in consumer memory), the replacement consumer resumes from the last committed offset, reprocessing at most a small overlap (at-least-once, not exactly-once, by default).
- Broker disk failure within a partition's replica set is absorbed by the ISR mechanism above; losing *all* replicas of a partition simultaneously is the only true data-loss scenario, which is why replication factor and failure-domain spread (different racks/AZs per replica) matter more than any single setting.

## 8. Wrap-up

Delivered: a leader/ISR replication design with an explicit, tunable durability-vs-availability knob, and a named failure mode (unclean leader election) that silently breaks the durability guarantee if left enabled. Not covered due to time: exactly-once semantics, schema evolution, and cross-datacenter replication.
