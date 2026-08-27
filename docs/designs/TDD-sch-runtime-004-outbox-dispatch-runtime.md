---
doc_meta:
  id: TDD-sch-runtime-004
  title: Scheduling Outbox and Dispatch Runtime
  owner: Scheduling Platform Team
  version: 1.0.0
  status: approved
  classification: restricted
  parent_sad: SAD-013
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-27
---
# Scheduling Outbox and Dispatch Runtime

## Purpose

Define crash-safe outbox relay behavior and the transport-neutral `OccurrenceDispatchPort` for Direct, RabbitMQ, and Kafka deployment profiles while preserving one logical Occurrence contract and one primary dispatch adapter per environment.

## Scope

Covers outbox leasing, dispatch attempts, durability acknowledgement, backoff/parking, logical CloudEvent envelope, RabbitMQ default topology, Kafka stream profile, Direct durable-acceptance profile, replay dispatch, and profile startup validation.

## Technical Context

Occurrence materialization has already committed a transport-neutral outbox record. Dispatch is at-least-once. `ACCEPTED` means the selected delivery boundary has durably accepted the trigger, never that the consumer completed business work.

RabbitMQ is the default production profile. Kafka is available when retained stream semantics are justified. Direct is allowed only for registered idempotent durable-acceptance targets.

## Component Design

```text
OutboxRelay
  -> OutboxRepository
  -> OccurrenceDispatchPort
      -> RabbitMQAdapter
      -> KafkaAdapter
      -> DirectAdapter
```

Exactly one primary adapter is enabled for `OccurrenceDue` in a normal environment. Broker/API client types terminate at adapters.

## Data Model

Outbox rows use the state machine:

`PENDING -> IN_FLIGHT -> ACCEPTED`

`IN_FLIGHT -> PENDING` occurs only after a lease expires or a retriable pre-acceptance failure is recorded.

Dispatch attempt evidence stores `attempt_no`, adapter profile, started/finished timestamps, normalized outcome, and safe error class. Occurrence `dispatch_state` becomes `ACCEPTED` only in the same local transaction that marks its outbox record accepted.

Logical CloudEvent data:

```json
{
  "specversion": "1.0",
  "type": "com.scnehaux.scheduling.occurrence.due.v1",
  "source": "urn:scnehaux:scheduling",
  "subject": "schedule/{schedule_id}",
  "id": "{dispatch_event_id}",
  "time": "{materialized_at}",
  "datacontenttype": "application/json",
  "data": {
    "occurrence_id": "...",
    "schedule_id": "...",
    "scheduled_for": "...",
    "application_id": "...",
    "tenant_id": "...",
    "target": {"contract": "...", "version": 1},
    "correlation_id": "...",
    "trigger": {}
  }
}
```

`dispatch_event_id` identifies one publication attempt/event; `occurrence_id` remains stable across retries and replay.

## API / Interface

```go
type OccurrenceDispatchPort interface {
    Dispatch(ctx context.Context, msg OccurrenceEnvelope) (DurableAcceptance, error)
}
```

The port returns success only after the profile-specific durability point.

RabbitMQ:
- durable direct/topic exchange `scnehaux.scheduling.occurrence.due.v1`
- persistent message
- mandatory routing
- registered target binding
- production queue type quorum
- publisher confirm required

Kafka:
- topic `scnehaux.scheduling.occurrence.due.v1`
- key `schedule_id`
- replicated producer acknowledgement required
- schema-compatible payload

Direct:
- POST to registered target acceptance URL from target registry
- `Idempotency-Key: <occurrence_id>`
- success only after target persisted/deduplicated the occurrence

## Algorithms / Logic

Relay claim:

1. Claim bounded `PENDING`/expired `IN_FLIGHT` rows with `FOR UPDATE SKIP LOCKED`
2. Set `IN_FLIGHT`, increment attempt, set lease, commit
3. Publish outside the DB transaction
4. On durable acceptance, mark outbox and Occurrence accepted
5. On known retriable pre-acceptance failure, schedule exponential backoff with full jitter
6. On poison/configuration failure, park and alert

Backoff starts at 250 ms, doubles to 30 s, and is capped; adapter-specific broker reconnect behavior remains below the port.

RabbitMQ publisher returns/negative confirms are failure. Kafka producer errors before required acknowledgement are failure. Direct 2xx is accepted only for a target contract that guarantees durable deduplicated acceptance.

Replay creates a new dispatch event referencing the same `occurrence_id` and marks `replay=true`; it does not create a new Occurrence.

## Configuration

Required startup configuration:

- `dispatch_profile = rabbitmq | kafka | direct`
- adapter-specific endpoint/credential references
- relay batch size default 200
- relay lease 30 s
- max concurrent publishes default 64
- parking threshold by error class

Startup fails if more than one primary profile is enabled for `OccurrenceDue`.

## Security Notes

Dispatch credentials come from secret delivery and are never persisted in Schedule/Occurrence/outbox payloads. Target routes are resolved from registered target metadata, not Schedule input. TLS and authenticated workload identity are mandatory outside local development.

Broker administrative APIs are not exposed through Scheduling Control API.

## Failure Handling

Ambiguous transport outcomes are treated conservatively. If the adapter cannot prove non-acceptance, it does not manufacture `ACCEPTED`; reconciliation determines whether retry is safe for that profile.

RabbitMQ/Kafka outages retain source outbox rows. Direct target outage retains source outbox rows. Poison/unroutable records park without blocking unrelated rows.

## Observability

Metrics:

- `scheduling_outbox_publish_total{profile,outcome}`
- `scheduling_outbox_publish_duration_seconds{profile}`
- `scheduling_outbox_lag_seconds`
- `scheduling_dispatch_lateness_seconds`
- `scheduling_dispatch_retries_total{profile}`
- `scheduling_dispatch_parked_total{reason}`
- `scheduling_replays_total`

Spans: `scheduling.outbox.claim`, `scheduling.dispatch`, adapter child span.

## Performance Notes

Relay concurrency is independent from due materialization. Backpressure reduces publish concurrency and eventually admission before source DB storage is exhausted. RabbitMQ is tuned for targeted queue delivery; Kafka profile is not enabled solely to increase throughput.

## Testing Strategy

Profile contract suite runs identically against Direct, RabbitMQ, and Kafka:

- accepted durability point
- broker/target outage
- process kill before/after publish
- duplicate publish preserving `occurrence_id`
- unroutable/poison message
- lease expiry
- replay identity
- startup rejection of dual primary adapters
- RabbitMQ publisher confirm and mandatory return
- Kafka acknowledgement/partition-key contract
- Direct idempotent durable acceptance

## Operational Notes

Dashboards distinguish source outbox backlog from broker/target backlog. Switching primary profile is a governed migration with drain/reconciliation; blind dual-publish is prohibited.

## Traceability

Implements SAD-013 Outbox Relay, Occurrence Dispatch Port, and dispatch adapters. Conforms to ADR-SCH-002 §§5.3-5.6 and STD-GLB-010 §§3.5-3.6, 3.11. Source outbox insertion is TDD-002.
