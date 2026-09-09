---
doc_meta:
  id: TDD-sch-runtime-005
  title: Scheduling Target Registry, Admission, and Quota
  owner: Scheduling Platform Team
  version: 1.2.0
  status: approved
  classification: restricted
  parent_sad: SAD-013
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-09-09
---
# Scheduling Target Registry, Admission, and Quota

## Purpose

Define registered Target projection, application/Tenant ownership enforcement, Schedule admission, quota accounting, due-dispatch fairness, and saturation policy so shared Scheduling cannot become an arbitrary executor or noisy-neighbor amplifier.

## Scope

Covers target metadata consumed by Scheduling, target enable/disable behavior, create-rate and active-Schedule quotas, due/replay quotas, privileged override, and degradation order. It excludes business authorization inside consumers and broker administration.

## Technical Context

Every Schedule targets a registered, versioned Product/Platform Target Contract. Arbitrary URL, shell command, function body, container image, or caller-selected unbounded priority is invalid. Target ownership is projected locally so normal due processing does not synchronously call another control plane.

Quotas are scoped by application and optional Tenant and protect creation, active cardinality, due rate, replay, and dispatcher concurrency.

## Component Design

```text
Control API -> TargetProjection -> Authorization
            -> AdmissionService -> QuotaRepository

DueRunner   -> FairnessPolicy
OutboxRelay -> TargetResolver
```

Target projection is refreshed asynchronously and is not authoritative for the external owning catalog. The projection carries lifecycle state, immutable contract version, compatibility policy, service class, replacement hints, and retirement window needed to enforce local scheduling safety. The last validated projection is sufficient for normal hot-path routing until its policy freshness limit is exceeded.

## Data Model

```sql
CREATE TABLE schedule_targets (
  target_id uuid PRIMARY KEY,
  contract_key text NOT NULL,
  contract_version integer NOT NULL,
  owner_application_id uuid NOT NULL,
  lifecycle_state text NOT NULL CHECK (lifecycle_state IN ('CANDIDATE','ACTIVE','DEPRECATED','RETIRED')),
  target_class text NOT NULL CHECK (target_class IN ('PRODUCT','PLATFORM','NOTIFICATION')),
  service_class text NOT NULL CHECK (service_class IN ('C1','C2','C3')),
  compatibility_policy text NOT NULL,
  replacement_target_id uuid NULL,
  deprecated_at timestamptz NULL,
  retire_after timestamptz NULL,
  allow_new_bindings_until timestamptz NULL,
  dispatch_route_ref text NOT NULL,
  projection_version bigint NOT NULL,
  validated_at timestamptz NOT NULL,
  UNIQUE (contract_key, contract_version)
);

CREATE TABLE schedule_quota_limits (
  application_id uuid NOT NULL,
  tenant_id uuid NULL,
  active_schedule_limit bigint NOT NULL,
  create_per_minute bigint NOT NULL,
  due_per_minute bigint NOT NULL,
  replay_per_minute bigint NOT NULL,
  max_dispatch_concurrency integer NOT NULL,
  UNIQUE NULLS NOT DISTINCT (application_id, tenant_id)
);

CREATE TABLE schedule_quota_usage_minute (
  bucket_start timestamptz NOT NULL,
  application_id uuid NOT NULL,
  tenant_id uuid NULL,
  creates bigint NOT NULL DEFAULT 0,
  dues bigint NOT NULL DEFAULT 0,
  replays bigint NOT NULL DEFAULT 0,
  UNIQUE NULLS NOT DISTINCT (bucket_start, application_id, tenant_id)
);
```

Active Schedule count is maintained transactionally on lifecycle changes in a compact quota counter, with reconciliation against authoritative Schedule rows.

The compact active-count authority is:

```sql
CREATE TABLE schedule_quota_counters (
  application_id uuid NOT NULL,
  tenant_id uuid NULL,
  active_schedules bigint NOT NULL CHECK (active_schedules >= 0),
  updated_at timestamptz NOT NULL DEFAULT transaction_timestamp(),
  UNIQUE NULLS NOT DISTINCT (application_id, tenant_id)
);
```

The counter is transactionally updated when schedules enter or leave active cardinality and is periodically reconciled against authoritative Schedule rows.

## API / Interface

Internal ports:

```go
type TargetResolver interface {
    Resolve(ctx context.Context, targetID uuid.UUID, owner Scope) (RegisteredTarget, error)
}
type AdmissionController interface {
    AdmitCreate(ctx context.Context, scope Scope) error
    AdmitReplay(ctx context.Context, scope Scope) error
}
```

Control API exposes read-only target discovery to normal consumers. Target mutation and quota override are privileged administration and require reason/evidence.

## Algorithms / Logic

Create admission:

1. Resolve exact Target Contract/version and verify ownership/lifecycle compatibility; `RETIRED` is never eligible, and `DEPRECATED` accepts new bindings only when its explicit compatibility window permits
2. Check active Schedule hard limit
3. Atomically increment current minute create bucket under limit
4. Create Schedule
5. Increment active count on successful commit

Due fairness exposes `eligible_schedule_scopes` to the due-claim query. Every accepted Schedule receives its service class from the registered Target profile (`C1 Critical`, `C2 Business`, `C3 Best Effort`); callers cannot elevate it. Class policy provides bounded reserved capacity/weights while application/Tenant quotas remain enforced inside each class. A scope is eligible only while its current due-rate budget and per-sweep fairness budget remain available. Materializing an Occurrence atomically consumes the scope's due bucket in the same authoritative transaction with a conditional increment (`UPDATE/UPSERT ... WHERE dues < due_limit RETURNING`). No returned row means the scope is no longer eligible for that minute. Concurrent replicas therefore cannot exceed the hard due budget through a check-then-act race. Relay workers additionally enforce bounded per-scope concurrency. The system never lets one scope or class occupy all claim or relay capacity. C1 capacity is protected during saturation, C2 receives normal business capacity, and C3 sheds/delays first; weighted fairness prevents indefinite starvation of lower classes.

Under saturation, degradation order is:

1. shed expensive administrative aggregations
2. reject preview/list work above concurrency budgets
3. constrain C3 Best Effort due work to its protected minimum and defer excess C3 backlog
4. reject new Schedule creates with 429 according to class/scope admission policy
5. preserve committed C1/C2 due materialization and outbox relay as long as database safety allows, while retaining bounded anti-starvation capacity for C3

Replay is lower priority than first-time dispatch.

## Configuration

Platform defaults are deployment configuration and can be overridden by governed scope policy:

- active Schedule limit
- create/minute
- due/minute
- replay/minute
- max dispatch concurrency
- target projection maximum age for new mutations
- class reservations/weights and bounded max concurrency for C1/C2/C3
- deprecation/new-binding window and retirement reconciliation grace period

No quota can be set to unbounded. Privileged operator override has a maximum duration and automatically expires.

## Security Notes

Target registration is privileged and binds contract to owning application/service identity. `dispatch_route_ref` is internal metadata and never caller-supplied in a Schedule request.

Cross-Tenant quota override requires step-up capable authority, explicit reason, expiry, and evidence. Quota error responses do not disclose other Tenant usage.

## Failure Handling

If quota/class state is unavailable, already committed due work continues under last safe local bounds while new create/replay admission fails closed. A Target cannot transition operationally to `RETIRED` while unresolved active Schedule bindings remain unless those bindings are explicitly grandfathered by a bounded policy. A stale target projection may continue dispatch for already accepted Schedules within the declared freshness policy, but new target changes/creates fail when safe ownership cannot be proven.

## Observability

Metrics:

- `scheduling_quota_utilization_ratio{dimension}`
- `scheduling_admission_rejected_total{reason}`
- `scheduling_active_schedules{scope}`
- `scheduling_target_projection_age_seconds`
- `scheduling_dispatch_concurrency{scope}`
- `scheduling_admin_shed_total`

Audit facts cover target change and quota override.

## Performance Notes

Minute buckets are compact and indexed by scope/time. Old buckets are retained only for the required evidence window then aggregated/expired. Quota checks use point reads/atomic updates, not full Schedule scans.

## Testing Strategy

Tests cover:

- target ownership mismatch
- target lifecycle transitions (`ACTIVE`/`DEPRECATED`/`RETIRED`) and blocked retirement with unresolved bindings
- arbitrary URL/command/priority rejection
- concurrent create quota race
- active-count reconciliation
- due fairness under one noisy scope
- C1/C2/C3 reserved-capacity fairness and C3 anti-starvation
- caller service-class escalation rejection
- replay deprioritization
- stale projection behavior
- privileged override expiry/evidence
- degradation order under load
- Tenant isolation

## Operational Notes

Operators view quota saturation and target freshness without direct DB access. Emergency override is time-bounded. Persistent quota pressure is capacity planning input, not a reason to silently disable fairness.

## Traceability

Implements SAD-013 v2.2 Target Contract lifecycle/versioning, Scheduling Service Classes, Target Projection, and Quota & Admission modules; conforms to PAD-PLT-011 §§5-7 and STD-GLB-010 §§3.2, 3.9-3.10. Dispatch route consumption is TDD-004.
