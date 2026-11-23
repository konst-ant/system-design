# Zero-Downtime Database Migration — System Design Solution

*Time-boxed to ~45–60 minutes. Diagrams: `ZeroDowntimeDatabaseMigration_component.excalidraw`, `ZeroDowntimeDatabaseMigration_sequence.excalidraw`. Part II — Architectural Evolution: a schema change (or engine/storage migration) must happen on a live, high-traffic database with no maintenance window.*

## 1. Problem & Scope

A large, continuously-written-to database needs a structural change (e.g. splitting one column into two, changing a column's type, or moving to a new storage engine) without stopping traffic, without a long maintenance window, and with the ability to back out safely if something goes wrong partway through.

**In scope:** a reversible, phased migration strategy; keeping both old and new schema consistent mid-migration; safe cutover and rollback.

**Explicitly deferred:** the specific mechanics of a storage-engine-level migration (e.g. moving between two different database products), downstream analytics/reporting pipeline migration, data-residency/compliance concerns during migration.

## 2. Current State & Targets

- **Current:** a 10 TB database serving ~20K QPS; the schema change under consideration (e.g. splitting a `name` column into `first_name`/`last_name`) cannot be done as a single blocking `ALTER TABLE` at this scale without unacceptable downtime.
- **Target:** complete the migration within a 4–6 week window, with zero user-facing downtime and a safe rollback available at every phase.

**Capacity check:** at 20K QPS, even a 60-second blocking schema change would queue roughly 1.2M requests — the entire point of this exercise is that "just run the migration" is not an option at this scale, which is why the strategy must be phased and reversible rather than a single cutover event.

## 3. Interface

```
(Unchanged client-facing API throughout — the migration is invisible to callers.)
Internal: feature flag migration_phase -> {expand, backfill, validate, contract}
```

## 4. High-Level Architecture — `ZeroDowntimeDatabaseMigration_component.excalidraw`

- **App Instances → Database**: write to both old and new schema during the transition (dual-write), gated by a feature flag per phase (§5).
- **CDC/Backfill Job → Database**: asynchronously copies and transforms existing rows from old schema to new schema, catching up historical data the dual-write doesn't cover.
- **Validation Job → Database**: compares old vs. new schema values for a sample (or all) rows to confirm correctness before cutover.
- **Feature Flag Service**: controls which phase is active — the single switch that can roll the migration forward or back.

## 5. Deep Dive — Expand-Contract (Parallel Change) — `ZeroDowntimeDatabaseMigration_sequence.excalidraw`

**Options considered:**
- A single blocking schema migration during a maintenance window: simplest to reason about, but requires downtime proportional to table size — unacceptable at 10 TB / 20K QPS, and this system has no maintenance window to begin with.
- Migrate by cutting over all at once after a background copy with no dual-write period: the background copy can never fully catch up while writes keep arriving, so there's always a last window of divergence between old and new data at cutover — exactly the gap this pattern is designed to close.
- **Pick: Expand-Contract (Parallel Change).** Each phase is independently reversible: *expand* (add the new schema alongside the old, nothing removed yet) → *backfill + dual-write* (new writes go to both; a background job fills in historical rows) → *validate* (confirm old and new agree) → *contract* (switch reads to the new schema, then remove the old one only once nothing depends on it).

**Terms:**

| Term | Meaning |
|---|---|
| Expand | Add the new schema element without removing or altering the old one — purely additive, so it can't break existing readers. |
| Dual-write | The application writes every change to both old and new schema simultaneously, keeping them in sync going forward. |
| Backfill | A background job that copies/transforms pre-existing rows into the new schema, since dual-write only covers writes *after* it's enabled. |
| Validate | Compare old vs. new schema data (sampled or exhaustive) to confirm the new schema is correct before anything depends on it exclusively. |
| Contract | Remove the old schema element, only after reads have fully moved to the new one and nothing references the old — the only phase that is not easily reversible. |

**Worked timeline:**

| Phase | Action | Rollback if needed |
|---|---|---|
| 1. Expand | Add `first_name`, `last_name` columns (nullable); `name` column untouched | Drop the new columns — no reader depends on them yet |
| 2. Dual-write | App writes update both `name` and `first_name`/`last_name` on every write going forward | Disable the flag — revert to writing `name` only |
| 3. Backfill | Background job walks existing rows, splits `name` into `first_name`/`last_name` for rows written before phase 2 | Re-run or pause the job; old `name` column is untouched throughout |
| 4. Validate | Sample 1% of rows, confirm `first_name + " " + last_name == name` for all sampled rows | Fix the transform logic and re-backfill the mismatched rows; nothing user-facing has changed yet |
| 5. Contract (reads) | Flip reads to use `first_name`/`last_name` instead of `name` | Flip the flag back to reading `name` — both are still being kept in sync by dual-write |
| 6. Contract (writes) | Stop writing `name`; drop the column | **Not reversible** — only done after phase 5 has run safely in production for a confidence period |

**Correctness pitfall:** treating phase 6 (dropping the old column) as safe to do as soon as phase 5 (reads flipped) looks fine is the main way this pattern fails in practice. Some caller — a report, an export job, another service with a cached schema assumption — may still read the old column after reads have nominally "moved." Dropping it immediately removes the only rollback path for phase 5. The old schema must be kept alive, in sync via dual-write, for a deliberate confidence period after cutover before the irreversible contract step runs — the same principle as never promoting a database change you can't undo until you've confirmed nothing depends on the state you're about to remove.

## 6. Data Model

```
users: {
  id, name (legacy, kept through phase 5),
  first_name (added phase 1), last_name (added phase 1)
}
migration_flags: { phase_name -> enabled_boolean }
backfill_progress: { last_processed_id, rows_migrated, rows_remaining }
```

## 7. Scaling, Failure & Trade-offs

- Dual-write doubles write-path work for the migrating columns during phases 2–5; at 20K QPS this is a measured, accepted cost for the multi-week migration window, not a permanent state.
- The backfill job runs at a throttled rate (batched by primary key range) to avoid competing with live traffic for the same resource that was already constrained — reusing the same diagnose-before-you-fix awareness of shared resource contention as `DatabaseBottleneckElimination.md` §5, just applied proactively here instead of reactively.
- Because every phase up to contract is reversible, the migration can safely pause for days or weeks between phases (e.g. waiting out the validation confidence period) without being "stuck" mid-migration — the phased structure is what makes a 4–6 week window tolerable for a change that is logically simple but risky to rush.

## 8. Wrap-up

Delivered: an Expand-Contract migration strategy that keeps every phase independently reversible except the final, deliberately-delayed contract step, enabling a schema change on a 10 TB / 20K QPS database with zero downtime. Not covered due to time: storage-engine-level migration mechanics, downstream analytics pipeline migration, and data-residency considerations.