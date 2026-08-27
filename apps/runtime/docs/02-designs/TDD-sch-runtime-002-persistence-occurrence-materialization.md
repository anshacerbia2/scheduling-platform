---
doc_meta:
  id: TDD-sch-runtime-002
  title: Scheduling Persistence and Occurrence Materialization
  owner: Scheduling Platform Team
  version: 1.0.0
  status: approved
  classification: restricted
  parent_sad: SAD-013
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-27
---
# Scheduling Persistence and Occurrence Materialization

## Purpose

Define the PostgreSQL schema, indexes, transaction boundaries, due-claim algorithm, Occurrence materialization, Schedule advancement, and crash-safe source outbox that make Scheduling temporally correct under multiple replicas.

## Scope

Covers authoritative runtime tables for Schedule, command idempotency, Occurrence, and publication outbox; exact uniqueness constraints; due acquisition; transaction isolation; mutation/materialization locking; and crash recovery.

Target registry/quota tables are defined in TDD-005. Transport publication after outbox commit is defined in TDD-004.

## Technical Context

PostgreSQL is the sole temporal authority. Host clocks, goroutine timers, RabbitMQ delayed delivery, Kafka timestamps, and in-memory cron registries are not authoritative. Database transaction time is canonical for determining due work.

All authoritative state mutation and publication intent occur in one local transaction. No network call is permitted while a Schedule row is locked.

## Component Design

```text
DueRunner
  -> DueScheduleRepository.claimDueBatch
      -> PostgreSQL
  -> OccurrenceMaterializer.materializeLocked
      -> TemporalCalculator
      -> OccurrenceRepository
      -> OutboxRepository
```

Each DueRunner replica executes short bounded transactions. `FOR UPDATE SKIP LOCKED` supplies cooperative multi-replica exclusion without a distributed lock service.

## Data Model

Core schema:

```sql
CREATE TABLE schedules (
  schedule_id uuid PRIMARY KEY,
  application_id uuid NOT NULL,
  tenant_id uuid NULL,
  target_id uuid NOT NULL,
  schedule_version bigint NOT NULL CHECK (schedule_version > 0),
  state text NOT NULL CHECK (state IN ('ACTIVE','PAUSED','COMPLETED','CANCELLED')),
  schedule_type text NOT NULL CHECK (schedule_type IN ('ONE_TIME','RECURRING')),
  one_time_at timestamptz NULL,
  recurrence_anchor_local timestamp without time zone NULL,
  recurrence_expression text NULL,
  recurrence_semantics_version text NOT NULL,
  time_zone text NULL,
  dst_policy_version text NOT NULL,
  dst_nonexistent text NULL,
  dst_ambiguous text NULL,
  misfire_policy text NULL,
  one_time_misfire_policy text NULL,
  max_catch_up integer NULL,
  next_due_at timestamptz NULL,
  created_at timestamptz NOT NULL DEFAULT transaction_timestamp(),
  updated_at timestamptz NOT NULL DEFAULT transaction_timestamp(),
  CHECK (
    (schedule_type='ONE_TIME' AND one_time_at IS NOT NULL AND recurrence_expression IS NULL)
    OR
    (schedule_type='RECURRING' AND one_time_at IS NULL AND recurrence_anchor_local IS NOT NULL AND recurrence_expression IS NOT NULL AND time_zone IS NOT NULL)
  )
);

CREATE INDEX schedules_due_idx
  ON schedules (next_due_at, schedule_id)
  WHERE state='ACTIVE' AND next_due_at IS NOT NULL;

CREATE TABLE schedule_command_idempotency (
  application_id uuid NOT NULL,
  tenant_id uuid NULL,
  idempotency_key text NOT NULL,
  semantic_fingerprint bytea NOT NULL,
  schedule_id uuid NOT NULL REFERENCES schedules(schedule_id),
  response_version bigint NOT NULL,
  created_at timestamptz NOT NULL DEFAULT transaction_timestamp(),
  UNIQUE NULLS NOT DISTINCT (application_id, tenant_id, idempotency_key)
);

CREATE TABLE occurrences (
  occurrence_id uuid PRIMARY KEY,
  schedule_id uuid NOT NULL REFERENCES schedules(schedule_id),
  schedule_version bigint NOT NULL,
  scheduled_for timestamptz NOT NULL,
  recurrence_semantics_version text NOT NULL,
  dst_policy_version text NOT NULL,
  time_zone text NULL,
  tzdata_version text NULL,
  materialized_at timestamptz NOT NULL DEFAULT transaction_timestamp(),
  dispatch_state text NOT NULL CHECK (dispatch_state IN ('PENDING','ACCEPTED')),
  dispatched_at timestamptz NULL,
  UNIQUE (schedule_id, scheduled_for)
);

CREATE TABLE scheduling_outbox (
  outbox_id uuid PRIMARY KEY,
  aggregate_type text NOT NULL,
  aggregate_id uuid NOT NULL,
  event_type text NOT NULL,
  payload jsonb NOT NULL,
  state text NOT NULL CHECK (state IN ('PENDING','IN_FLIGHT','ACCEPTED')),
  attempt_count integer NOT NULL DEFAULT 0,
  available_at timestamptz NOT NULL DEFAULT transaction_timestamp(),
  lease_until timestamptz NULL,
  accepted_at timestamptz NULL,
  created_at timestamptz NOT NULL DEFAULT transaction_timestamp()
);
CREATE INDEX scheduling_outbox_pending_idx
  ON scheduling_outbox (available_at, created_at)
  WHERE state <> 'ACCEPTED';
```

The schema requires PostgreSQL 15+ semantics for `UNIQUE NULLS NOT DISTINCT`, allowing non-Tenant idempotency without a magic sentinel Tenant identifier.

Production RLS policies scope `schedules`, `occurrences`, and command-recovery views to authenticated application/Tenant context. Internal due/outbox roles use narrowly privileged bypass roles unavailable to API callers.

## API / Interface

Repository ports expose transaction-scoped operations only:

```go
type ScheduleTxRepository interface {
    ClaimDue(ctx context.Context, limit int) ([]LockedSchedule, error)
    InsertOccurrence(ctx context.Context, o Occurrence) error
    AdvanceSchedule(ctx context.Context, next *time.Time, terminal bool) error
    InsertOutbox(ctx context.Context, e OutboxRecord) error
}
```

The materializer receives a locked Schedule and must complete all occurrence/advance/outbox mutations before commit.

## Algorithms / Logic

The physical due query includes the eligibility predicate supplied by TDD-005 so scopes that have exhausted their due-rate/fairness budget are not repeatedly selected ahead of other scopes. The core locking shape is:

```sql
SELECT s.*
FROM schedules s
JOIN eligible_schedule_scopes e
  ON e.application_id = s.application_id
 AND e.tenant_id IS NOT DISTINCT FROM s.tenant_id
WHERE s.state='ACTIVE'
  AND s.next_due_at IS NOT NULL
  AND s.next_due_at <= transaction_timestamp()
ORDER BY s.next_due_at, s.schedule_id
FOR UPDATE OF s SKIP LOCKED
LIMIT $1;
```

`eligible_schedule_scopes` is a transaction-local/query CTE or view-like relation derived from current quota/fairness state; it is not a second scheduling authority.

For each locked Schedule inside the same transaction:

1. Re-evaluate due state from persisted version and database transaction time
2. Apply misfire policy using the pure calculator
3. Insert each permitted Occurrence with UUIDv7
4. Insert one `occurrence.due` outbox record per Occurrence
5. Advance `next_due_at`, or mark one-time Schedule `COMPLETED`
6. Commit

The unique `(schedule_id, scheduled_for)` constraint is the final duplicate-creation guard.

Pause/update/cancel takes the same Schedule row lock. Therefore the transaction commit order implements the SAD linearization rule without a separate distributed lock.

## Configuration

- `due_batch_size`: default 100, range 1..1000
- `due_poll_interval`: default 250 ms, range 50 ms..5 s
- `statement_timeout`: 5 s for due transactions
- `lock_timeout`: 1 s for administrative mutation
- `outbox_payload_max_bytes`: 64 KiB

Values are environment configuration with safe upper bounds. They do not change Schedule semantics.

## Security Notes

Runtime DB role has DML only and no DDL. Migration role is separate. RLS is enabled for Tenant-scoped API/query access. DueRunner and OutboxRelay use dedicated service roles with only the tables/columns needed by their function.

Outbox payload contains bounded trigger data and no credentials. SQL statements are parameterized. Application-provided recurrence text is stored only after TDD-001 validation.

## Failure Handling

- crash before transaction commit: no Occurrence, Schedule advance, or outbox effect is visible
- crash after commit: committed Occurrence and outbox remain discoverable
- concurrent replica: locked row is skipped; unique constraint prevents duplicate logical Occurrence
- database failover: transaction outcome is recovered by normal retry and persisted state inspection
- outbox outage: due materialization continues until configured storage/backpressure thresholds require admission reduction; accepted state is not lost

## Observability

Emit:

- `scheduling_due_claim_duration_seconds`
- `scheduling_due_claimed_schedules_total`
- `scheduling_occurrences_materialized_total`
- `scheduling_db_lock_wait_seconds`
- `scheduling_db_serialization_retries_total`
- `scheduling_outbox_rows{state}`
- `scheduling_outbox_oldest_pending_seconds`

Trace spans: `scheduling.db.claim_due`, `scheduling.materialize`, `scheduling.db.commit_due`.

## Performance Notes

The due index is partial and ordered by `next_due_at`. The transaction processes a bounded batch and performs no network I/O. Capacity certification must demonstrate 10x forecast peak due rate while p99 claim transactions remain below 500 ms and dispatch-lateness SLO remains satisfied.

Partitioning is not enabled initially. It is introduced only when measured table size/vacuum/index behavior warrants it.

## Testing Strategy

Blocking integration tests run against production-equivalent PostgreSQL:

- 2, 8, and 32 concurrent DueRunner replicas
- unique Occurrence under concurrent claims
- kill process before and after commit
- update/pause/cancel racing materialization
- catch-up batch boundaries
- one-time completion
- RLS isolation under runtime API role
- migration/runtime role privilege separation
- index-plan assertions for due and outbox scans
- database restart/failover recovery
- 10x forecast load and lock-contention profiling

## Operational Notes

Vacuum/analyze health, table growth, due-index bloat, transaction age, and outbox lag are dashboarded. Operators never repair schedules by direct SQL in normal operations; governed reconciliation APIs perform supported repair.

Backups include Schedule, Occurrence, idempotency, and outbox state. HA commit acknowledgment provides RPO=0 inside the declared HA failure domain.

## Traceability

Implements SAD-013 State & Data Architecture and Due Claimer/Occurrence Materializer modules. Conforms to PAD-PLT-011, ADR-SCH-002 §§5.1-5.2, and STD-GLB-010 §§3.5, 3.8, 3.11. Temporal rules come from TDD-001; publication relay is TDD-004.
