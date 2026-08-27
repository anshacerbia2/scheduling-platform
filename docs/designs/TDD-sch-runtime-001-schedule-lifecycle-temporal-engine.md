---
doc_meta:
  id: TDD-sch-runtime-001
  title: Schedule Lifecycle and Temporal Engine
  owner: Scheduling Platform Team
  version: 1.1.0
  status: approved
  classification: restricted
  parent_sad: SAD-013
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-28
---
# Schedule Lifecycle and Temporal Engine

## Purpose

Define the authoritative Schedule aggregate, mutation state machine, recurrence contract, daylight-saving-time behavior, misfire behavior, and deterministic temporal calculation used by Scheduling Runtime. This design fixes observable temporal semantics before persistence and transport implementation so later adapter changes cannot alter Schedule meaning.

## Scope

Covers one-time and recurring Schedule definitions, versioning, preview, pause/resume/update/cancel, completion, recurrence semantics, DST semantics, misfire selection, and the linearization rule between mutation and Occurrence materialization.

Excludes PostgreSQL DDL and claim SQL, transport adapters, target registry, quota, and browser UX. Those are owned by sibling TDDs under SAD-013 and SAD-014.

## Technical Context

SAD-013 makes PostgreSQL the temporal authority and requires a pure Temporal Calculator. STD-GLB-010 requires canonical UTC Occurrences, IANA time-zone identifiers for wall-clock recurrence, versioned recurrence interpretation, versioned DST behavior, explicit misfire, stable Occurrence identity, and deterministic near-due mutation semantics.

The contract uses two time models:

- one-time schedules provide one canonical RFC 3339 UTC instant
- recurring wall-clock schedules provide a local wall-time anchor, a versioned recurrence expression, and an IANA `time_zone`

The platform semantic identifier is `scnehaux-rfc5545-v1`. It is an RFC 5545-compatible governed subset supporting `FREQ` (`DAILY`, `WEEKLY`, `MONTHLY`, `YEARLY`), `INTERVAL`, `BYDAY`, `BYMONTHDAY`, `BYMONTH`, `COUNT`, and UTC `UNTIL`. `DTSTART` is represented by the separate `recurrence_anchor_local` field. `BYSECOND`, `BYMINUTE`, `BYHOUR`, `BYSETPOS`, `BYWEEKNO`, `BYYEARDAY`, `WKST`, and extension properties are rejected in v1. Unsupported features fail validation instead of being interpreted differently by different libraries.

## Component Design

The runtime contains a dependency-free `domain/schedule` package and a replaceable `app/temporal` calculator.

```text
Control API
  -> ScheduleCommandService
      -> Schedule aggregate
      -> TemporalCalculator
      -> ScheduleRepository port

Due Runner
  -> OccurrenceMaterializer
      -> Schedule aggregate
      -> TemporalCalculator
```

`Schedule` states are `ACTIVE`, `PAUSED`, `COMPLETED`, and `CANCELLED`.

`CANCELLED` and `COMPLETED` are terminal. A recurring Schedule normally remains `ACTIVE`; a one-time Schedule becomes `COMPLETED` after its only Occurrence is materialized. Pause preserves the recurrence definition and current version while preventing future materialization.

Every semantic mutation increments `schedule_version`. Pure metadata that does not alter scheduling behavior is stored separately and does not silently change the temporal version.

### Authoritative State Transition Contract

| Current | Event | Next | Guard |
| --- | --- | --- | --- |
| none | create | `ACTIVE` | valid temporal spec and admitted target |
| `ACTIVE` | pause | `PAUSED` | expected version matches |
| `PAUSED` | resume | `ACTIVE` or `COMPLETED` | persisted misfire policy applied |
| `ACTIVE` | semantic update | `ACTIVE` | only future non-materialized occurrences affected |
| `PAUSED` | semantic update | `PAUSED` | no occurrence materialization |
| `ACTIVE`/`PAUSED` | cancel | `CANCELLED` | expected version matches |
| `ACTIVE` one-time | materialize occurrence | `COMPLETED` | materialization transaction commits |
| `COMPLETED`/`CANCELLED` | any mutation | rejected | terminal |

Duplicate no-op commands may return current state only when idempotency proves semantic equivalence and MUST NOT increment `schedule_version`. Pause, update, cancel, and materialization serialize through the same Schedule row lock.

## Data Model

The aggregate exposes these immutable/versioned fields to persistence:

| Field | Rule |
| --- | --- |
| `schedule_id` | UUIDv7, immutable |
| `application_id` | immutable owner |
| `tenant_id` | nullable only for non-Tenant scope |
| `target_id` | registered target reference |
| `schedule_version` | positive integer, increments on semantic mutation |
| `schedule_type` | `ONE_TIME` or `RECURRING` |
| `one_time_at` | UTC instant for one-time |
| `recurrence_anchor_local` | local wall datetime without UTC offset; required for recurring |
| `recurrence_expression` | governed expression for recurring |
| `recurrence_semantics_version` | `scnehaux-rfc5545-v1` |
| `time_zone` | IANA identifier for recurring |
| `dst_policy_version` | `dst-v1` |
| `dst_nonexistent` | `SKIP` or `SHIFT_FORWARD` |
| `dst_ambiguous` | `EARLIER_OFFSET` or `LATER_OFFSET` |
| `misfire_policy` | recurring: `SKIP`, `FIRE_ONCE`, `CATCH_UP_BOUNDED` |
| `one_time_misfire_policy` | `SKIP` or `FIRE_ONCE` |
| `max_catch_up` | required only for `CATCH_UP_BOUNDED`, 1..100 |
| `next_due_at` | next canonical UTC candidate |
| `state` | lifecycle state |

Materialized Occurrences retain `schedule_version`, `recurrence_semantics_version`, `dst_policy_version`, `time_zone`, and the effective tzdata version as immutable computation evidence.

## API / Interface

Internal ports:

```go
type TemporalCalculator interface {
    Preview(spec ScheduleSpec, after time.Time, limit int) ([]ComputedOccurrence, error)
    Next(spec ScheduleSpec, after time.Time) (*ComputedOccurrence, error)
    Missed(spec ScheduleSpec, from, through time.Time, max int) ([]ComputedOccurrence, error)
}
```

`ComputedOccurrence` contains `ScheduledForUTC`, `LocalWallTime`, `UTCOffsetSeconds`, and `TZDataVersion`.

Domain commands are `Create`, `UpdateTemporalPolicy`, `Pause`, `Resume`, and `Cancel`. Every mutation receives `ExpectedVersion`; stale versions return a conflict and never auto-merge.

## Algorithms / Logic

**Creation**

1. Validate ownership/target outside the domain
2. Normalize the temporal specification
3. Compute and persist `next_due_at`
4. Start at version 1 and `ACTIVE`

**DST**

For a nonexistent wall time, `SKIP` omits that logical occurrence. `SHIFT_FORWARD` adds the exact UTC-offset gap (`offset_after - offset_before`) to the nonexistent local wall time; for a one-hour spring gap, 02:30 becomes 03:30. For an ambiguous repeated wall time, `EARLIER_OFFSET` or `LATER_OFFSET` deterministically selects one of the two valid UTC instants. DST behavior is part of the Schedule version.

**Resume and outage recovery**

For elapsed recurring time:
- `SKIP` advances directly to the first future occurrence
- `FIRE_ONCE` returns one recovery occurrence whose `scheduled_for` is the latest missed logical instant
- `CATCH_UP_BOUNDED` returns at most `max_catch_up` missed instants in ascending order

For overdue one-time schedules:
- `SKIP` completes without materializing an Occurrence
- `FIRE_ONCE` materializes the original one-time `scheduled_for`

**Mutation linearization**

The persistence transaction that materializes an Occurrence is the linearization point. A mutation committed before materialization prevents the previous Schedule version from materializing that future Occurrence. A materialization committed first remains valid and immutable.

## Configuration

The runtime configuration exposes bounded defaults only:

- `preview_max_occurrences = 100`
- `catch_up_absolute_max = 100`
- `recurrence_semantics_version = scnehaux-rfc5545-v1`
- `dst_policy_version = dst-v1`

A recurring Schedule cannot rely on server-local timezone or an implicit DST policy. Configuration rollout never rewrites existing Schedule versions.

## Security Notes

Temporal input is treated as untrusted. Expression length, token count, preview count, and date horizon are bounded before calculation to prevent parser or CPU abuse. Callers cannot supply executable code, provider credentials, arbitrary callback URLs, or database expressions.

Ownership and authorization are enforced by the Control API TDD; the domain accepts an already-authorized ownership context.

## Failure Handling

A calculation failure does not partially mutate the Schedule. Unsupported recurrence syntax, invalid timezone, impossible policy combination, or preview overflow returns a deterministic validation error.

A tzdata upgrade that changes future computed UTC instants is a compatibility event. Deployment is blocked until the golden corpus and differential comparison identify the affected Schedule versions and the rollout evidence is accepted.

## Observability

Emit:

- `scheduling_temporal_calculation_duration_seconds`
- `scheduling_temporal_calculation_errors_total{reason}`
- `scheduling_misfire_recovery_total{policy}`
- `scheduling_tzdata_differential_changes_total`
- span `scheduling.temporal.preview`
- span `scheduling.temporal.next`

Logs contain Schedule IDs and policy versions but no trigger payload values.

## Performance Notes

Temporal calculation is pure CPU work and must not perform network or database calls. Preview is capped at 100 results. Due processing calculates only the bounded number required for the current transaction and misfire policy.

Golden/load tests certify recurrence calculation well below the database claim-transaction budget so long row locks are not introduced.

## Testing Strategy

Blocking tests include:

- one-time UTC creation and completion
- recurring daily/weekly/monthly/calendar-boundary cases
- DST gap under both nonexistent-time policies
- DST overlap under both ambiguous-time policies
- leap-day and month-end behavior
- `SKIP`, `FIRE_ONCE`, and bounded catch-up
- overdue one-time `SKIP` and `FIRE_ONCE`
- pause/resume/update/cancel state transitions
- stale `ExpectedVersion`
- mutation versus materialization race contract
- tzdata upgrade golden/differential corpus
- fuzz tests for recurrence parser normalization and bounded complexity

## Operational Notes

Operators can preview the exact future instants and the policy/tzdata evidence that produced them. Existing Schedule versions are never silently reinterpreted during incident repair.

If a future requirement needs sub-second precision, alternate calendar semantics, or regional temporal authority, it requires an explicit architecture profile rather than extending `scnehaux-rfc5545-v1` incompatibly.

## Traceability

Implements SAD-013 Temporal Calculator and Schedule Domain. Conforms to PAD-PLT-011, STD-GLB-010 §§3.3-3.8, and ADR-SCH-002 §§5.1, 5.2, and 5.7. Persistence linearization is realized by `TDD-sch-runtime-002`; HTTP mutation semantics are realized by `TDD-sch-runtime-003`.
