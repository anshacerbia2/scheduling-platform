---
doc_meta:
  id: TDD-sch-runtime-003
  title: Scheduling Control API and Command Idempotency
  owner: Scheduling Platform Team
  version: 1.0.0
  status: approved
  classification: restricted
  parent_sad: SAD-013
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-27
---
# Scheduling Control API and Command Idempotency

## Purpose

Define the versioned Scheduling Control API, authentication-derived ownership, command idempotency, optimistic concurrency, recovery after ambiguous create responses, pagination, and RFC 9457 error semantics.

## Scope

Covers REST control/query endpoints for Schedule lifecycle, preview, Occurrence query, replay request, target discovery, and reconciliation. It does not define browser session behavior, provider transports, or physical target registration administration.

## Technical Context

The API is a protected server-to-server resource consumed by Products/Platforms and Scheduling Experience BFF. Successful create is returned only after Schedule and idempotency state commit. Caller-supplied ownership identifiers never override authenticated workload/application and Tenant context.

## Component Design

```text
HTTP Adapter
  -> AuthN/AuthZ Middleware
  -> Idempotency Middleware
  -> Command Service / Query Service
  -> Domain + Repositories
```

Request DTOs are adapter types. Domain packages do not import HTTP, OIDC, JSON, or RFC 9457 packages.

## Data Model

Create semantic fingerprint is SHA-256 over canonical JSON of all fields that affect Schedule identity/behavior:

`target_id`, schedule type/spec including recurring wall-time anchor, timezone/DST/misfire policy, bounded trigger data, and authenticated ownership scope.

Presentation-only fields and request correlation IDs are excluded. Canonicalization uses sorted object keys, normalized RFC3339 UTC instants, and normalized recurrence syntax.

List/query cursors are opaque signed encodings of `(sort_key, schedule_id, filter_hash)` and expire after 24 hours.

## API / Interface

Base path `/v1`.

| Method | Path | Semantics |
| --- | --- | --- |
| POST | `/schedules` | idempotent create; requires `Idempotency-Key` |
| GET | `/schedules/{schedule_id}` | owned detail |
| GET | `/schedules` | cursor list/filter |
| PATCH | `/schedules/{schedule_id}` | semantic update; requires `If-Match` |
| POST | `/schedules/{schedule_id}:pause` | pause; `If-Match` |
| POST | `/schedules/{schedule_id}:resume` | resume; `If-Match` |
| POST | `/schedules/{schedule_id}:cancel` | terminal cancel; `If-Match` |
| POST | `/schedule-previews` | server-authoritative preview |
| GET | `/occurrences/{occurrence_id}` | occurrence evidence |
| GET | `/occurrences` | bounded occurrence query |
| POST | `/occurrences/{occurrence_id}:replay` | privileged replay request |
| GET | `/targets` | authorized target discovery |
| GET | `/idempotency/{key}` | owned create-result recovery |
| POST | `/reconciliations` | privileged/owned reconciliation |

Create returns `201 Created` with `schedule_id`, version, state, and next occurrence. Equivalent retries with the same key return `200 OK` and the same Schedule. Conflicting reuse returns `409`.

Mutation ETag is `"schedule-v<N>"`; stale `If-Match` returns `412 Precondition Failed`.

Errors are `application/problem+json` per RFC 9457 with stable `type`, `status`, `code`, `correlation_id`, and safe field violations.

## Algorithms / Logic

Create flow:

1. Authenticate and derive application/Tenant scope
2. Validate target authorization and quota
3. Normalize request and compute semantic fingerprint
4. In one DB transaction, lock/create the scoped idempotency row
5. If existing fingerprint matches, return existing Schedule
6. If existing fingerprint differs, return conflict
7. Otherwise create Schedule and persist idempotency mapping
8. Commit before responding

Mutation flow verifies ownership, locks the Schedule, checks ETag version, applies domain command, increments version for semantic change, writes lifecycle outbox, and commits.

Recovery endpoint never searches across ownership boundaries and never creates a new Schedule.

## Configuration

- `max_page_size = 200`
- `default_page_size = 50`
- `max_trigger_data_bytes = 32 KiB`
- `idempotency_key_max_length = 200`
- `request_body_max_bytes = 128 KiB`
- control timeout 5 s
- preview timeout 3 s

API versioning is path-based for major versions and additive within a major version.

## Security Notes

Access tokens are audience-bound and validated locally. Authorization derives `application_id` and canonical Tenant context from trusted claims/projections. Cross-Tenant provider administration is a separate privileged route/policy and requires reason/evidence.

Idempotency lookup is scoped to authenticated ownership. Object IDs are non-enumerable UUIDv7. Rate limits apply before expensive preview calculations.

## Failure Handling

Lost response after commit is resolved by retrying the same `Idempotency-Key` or using the recovery endpoint. A network retry cannot create a second logical Schedule.

Database timeout before known commit outcome is treated as ambiguous; the client retries with the same key. Validation/auth errors are not retried automatically.

## Observability

Emit request count/latency/error metrics by route and status, plus:

- `scheduling_idempotency_replays_total`
- `scheduling_idempotency_conflicts_total`
- `scheduling_optimistic_conflicts_total`
- `scheduling_reconciliation_requests_total`

Trace attributes include route, application/Tenant scope hashes, Schedule ID, and correlation ID; trigger data is excluded.

## Performance Notes

Read APIs use indexed server-side pagination. Preview and list queries have strict bounds. Administrative aggregation traffic has separate concurrency limits so it cannot starve due processing.

## Testing Strategy

Contract tests cover:

- create and equivalent retry
- lost-response retry
- conflicting key reuse
- unauthorized ownership spoof
- stale ETag on every mutation
- cursor tampering/expiry
- replay privilege and reason requirement
- RFC 9457 schema
- request/body/preview bounds
- Tenant isolation negative paths
- OpenAPI backward-compatibility checks

## Operational Notes

OpenAPI is generated from or validated against this contract and published under `packages/contracts/openapi/`. Breaking changes require a new major API version. Operational repair is exposed through reconciliation APIs rather than direct DB access.

## Traceability

Implements SAD-013 Control API and command idempotency requirements; conforms to PAD-PLT-011 ownership/reconciliation policy and STD-GLB-010 §§3.2, 3.8, 3.12. Persistence mapping is TDD-002; quota/target authorization is TDD-005.
