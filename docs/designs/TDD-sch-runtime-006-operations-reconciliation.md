---
doc_meta:
  id: TDD-sch-runtime-006
  title: Scheduling Operations and Reconciliation
  owner: Scheduling Platform Team
  version: 1.1.0
  status: approved
  classification: restricted
  parent_sad: SAD-013
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-28
---
# Scheduling Operations and Reconciliation

## Purpose

Define supported operational queries, reconciliation, replay governance, repair boundaries, SLI/SLO measurement, alerts, and incident evidence for Scheduling Runtime.

## Scope

Covers operator-visible runtime health, Schedule/Occurrence reconciliation, safe replay, idempotency-result recovery, stuck outbox repair, quota/target evidence, runbooks, and service-level indicators. It does not authorize direct manual mutation of authoritative tables.

## Technical Context

Scheduling owns temporal and dispatch state but not consumer business completion. Operational tooling must therefore show the boundary clearly and cannot infer Product success from dispatch acceptance.

## Component Design

`OperationsQueryService` reads bounded projections. `ReconciliationService` compares internal invariants and performs only enumerated repair commands. `ReplayService` delegates to outbox dispatch using the existing `occurrence_id`.

All repair commands are idempotent, reasoned, audited, and expose a dry-run mode.

## Data Model

Operational evidence references:

- Schedule ID/version/state
- Occurrence ID/scheduled_for/materialized_at
- dispatch attempts and durability point
- outbox state/age
- target/projection version
- recurrence/DST/tzdata evidence
- replay generation/reason/operator
- reconciliation run ID and findings

Reconciliation findings use stable codes such as `OUTBOX_MISSING`, `OCCURRENCE_PENDING_TOO_LONG`, `SCHEDULE_NEXT_DUE_INCONSISTENT`, and `QUOTA_COUNTER_DRIFT`.

## API / Interface

Privileged endpoints:

- `POST /v1/reconciliations` with scope and dry-run
- `GET /v1/reconciliations/{id}`
- `POST /v1/occurrences/{id}:replay`
- `POST /v1/operations/outbox/{id}:retry` for known-safe unpublished state
- read-only health/backlog endpoints consumed by Experience BFF

No endpoint can fabricate a new Occurrence to repair a dispatch problem.

## Algorithms / Logic

Reconciliation checks:

1. every pending Occurrence has its expected publication lineage
2. accepted outbox implies Occurrence dispatch accepted
3. terminal Schedule has no future `next_due_at`
4. active Schedule next due is calculator-consistent within its policy version
5. active-count quota counters reconcile to authoritative state
6. stale in-flight leases are recoverable

Replay creates a new publication intent referencing the same Occurrence and records replay generation/reason. It does not alter `scheduled_for`.

### Repair Contract

| Finding | Supported action |
| --- | --- |
| expired `IN_FLIGHT` outbox lease | return to `PENDING` preserving attempt evidence |
| `PARKED` outbox | no automatic redrive |
| active-count drift | recalculate scoped counter from authoritative Schedules |
| accepted outbox but Occurrence not accepted | repair projection from accepted outbox evidence |
| missing outbox for committed pending Occurrence | create one replacement intent only when evidence proves it is missing |
| temporal inconsistency | report only; never rewrite Schedule policy automatically |

Every mutating repair records pre/post state hashes, finding code, actor/service identity, reason, and reconciliation run ID.

## Configuration

- SLO lateness objective: 99.9% <= 30 s
- reconciliation scan page: 500
- automatic reconciliation concurrency: low-priority bounded pool
- oldest-pending warning: 15 s
- oldest-pending critical: 30 s sustained 5 min, excluding declared substrate outage
- error-budget window: rolling 30 days

Alert thresholds are deployment-profile aware.

## Security Notes

Replay, repair, quota override, target change, and cross-Tenant reconciliation require privileged authorization and evidence. Raw trigger data is redacted from general operational views. Support roles receive least-privilege read scopes.

## Failure Handling

Reconciliation itself never holds long locks or performs unbounded scans. A failed repair leaves authoritative state unchanged unless its local transaction commits. Unsupported inconsistency is escalated rather than fixed by ad-hoc SQL.

A declared broker outage suppresses duplicate symptoms but does not suppress source outbox growth/storage safety alerts.

## Observability

Canonical metrics:

- `scheduling_active_schedules`
- `scheduling_occurrences_materialized_total`
- `scheduling_dispatch_lateness_seconds`
- `scheduling_oldest_undispatched_seconds`
- `scheduling_dispatch_total{outcome}`
- `scheduling_misfires_total{policy}`
- `scheduling_replays_total`
- `scheduling_duplicate_dispatch_total`
- `scheduling_quota_utilization_ratio`
- `scheduling_due_claim_duration_seconds`
- `scheduling_outbox_lag_seconds`

Service SLI uses durable dispatch timestamp minus `scheduled_for`; Product execution time is excluded.

## Performance Notes

Operational scans are paged, rate-limited, and lower priority than due materialization/relay. Long-range history queries use bounded windows and indexes; dashboards consume aggregates where available.

## Testing Strategy

Tests inject:

- missing/late outbox publication
- stale relay leases
- quota counter drift
- target projection staleness
- replay under duplicate delivery
- cross-Tenant authorization failures
- reconciliation process crashes
- SLO alert boundary cases
- declared broker outage behavior
- runbook exercises for DB and broker failover

## Operational Notes

Runbooks cover DB failover, broker outage, relay backlog, due-lateness breach, tzdata compatibility rollout, stuck reconciliation, quota saturation, and target disablement.

The platform is not labeled battle-tested until fault/load exercises and production SLO evidence validate these runbooks.

### Backup and Restore Acceptance

A restore is healthy only after Schedule/idempotency/Occurrence/outbox reconciliation, pending/parked discovery, target/quota projection verification, preserved `occurrence_id` replay, and measured RPO/RTO. At least one production-like restore drill is required before calling the platform operationally mature.

## Traceability

Implements SAD-013 Operations & Reconciliation and NFR sections. Conforms to PAD-PLT-011 §6 and STD-GLB-010 §3.12. Replay dispatch is TDD-004; persistence invariants are TDD-002.
