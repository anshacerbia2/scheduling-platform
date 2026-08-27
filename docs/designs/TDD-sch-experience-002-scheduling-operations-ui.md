---
doc_meta:
  id: TDD-sch-experience-002
  title: Scheduling Operations User Interface
  owner: Scheduling Platform Team
  version: 1.0.0
  status: approved
  classification: restricted
  parent_sad: SAD-014
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-27
---
# Scheduling Operations User Interface

## Purpose

Define React feature boundaries, server-authoritative recurrence preview, lifecycle mutation UX, Occurrence timeline, replay/misfire workflows, quota/target visibility, accessibility, and client state invalidation.

## Scope

Covers browser presentation and interaction only. All authorization, temporal calculation, lifecycle state, replay, quota, and target authority remain server-side.

## Technical Context

The UI consumes generated/validated BFF contracts and Scnehaux UI Platform primitives. It is organized by Scheduling concepts rather than RabbitMQ/Kafka/PostgreSQL implementation concepts.

## Component Design

Feature modules:

- `schedule-management`
- `schedule-editor`
- `occurrence-timeline`
- `misfire-replay`
- `quota-usage`
- `target-discovery`
- `operational-health`
- `audit-correlation`

No feature imports Runtime internal source. Shared UI state is limited to session/context, routing, notifications, and bounded query cache.

## Data Model

Client models are generated from the governed Scheduling API contract. Form draft state is ephemeral. Mutation records include current ETag and `context_generation`.

Time display always carries:
- user-facing local time and zone
- canonical UTC instant on detail/confirmation screens
- DST/misfire policy where recurrence is edited

## API / Interface

UI calls only `/api/scheduling/*` BFF routes.

Create/update forms request server preview before final activation when recurrence changes. Destructive actions display Schedule, Tenant, Application, target, and effective version. Replay requires explicit reason and step-up-capable confirmation.

## Algorithms / Logic

On context change, cancel all old-generation requests and destroy query cache, mutation queue, selected rows, and form drafts before rendering the new scope.

Mutation conflict (`412`) never auto-overwrites; the UI fetches current state and presents a comparison/retry choice.

Replay UI states that the same `occurrence_id` is re-dispatched and does not imply a new business occurrence.

## Configuration

- default list page 50
- timeline default window 7 days, max 90 days per query
- recurrence preview default 10, max 100
- client request timeout follows BFF
- accessibility target WCAG 2.2 AA

## Security Notes

Hidden/disabled controls are presentation only. Privileged actions remain server-authorized. Trigger payload display is redacted by contract and never rendered as unrestricted raw JSON to general operators.

## Failure Handling

Partial query failures preserve usable sections and show correlation IDs. Duplicate submit reuses the same idempotency identity generated for that form submission. Unsaved form state is intentionally discarded on context switch.

## Observability

Browser telemetry records route, feature, outcome, Web Vitals, and correlation IDs only. It excludes token/session values and sensitive trigger fields. Accessibility failures are CI artifacts rather than runtime telemetry.

## Performance Notes

Large tables use server pagination and windowed rendering. No screen loads unbounded Occurrence history. Bundle/performance budgets are enforced per feature.

## Testing Strategy

E2E tests cover create/preview/update/pause/resume/cancel, stale ETag, DST preview parity, context switch, duplicate submit, replay confirmation, quota visualization, partial API failure, keyboard navigation, focus management, semantic status feedback, and WCAG 2.2 AA automated/manual checks.

## Operational Notes

UI release/rollback is independent. A broker profile change requires no UI change while Scheduling contracts remain compatible. Feature flags may hide unfinished presentation, but cannot alter Runtime semantics.

## Traceability

Implements SAD-014 Schedule Management, Occurrence Timeline, Misfire & Replay, Usage & Quota, Target Discovery, Operational Health, and Audit Correlation capabilities. Security boundary is TDD-sch-experience-001.
