# Scheduling Platform

A reusable platform with independently deployable Runtime and Experience applications.

## Repository Topology

```text
scheduling-platform/
├─ apps/
│  ├─ runtime/
│  └─ experience/
│     ├─ server/
│     └─ web/
├─ packages/
│  └─ contracts/
├─ docs/
│  └─ designs/
└─ deploy/
```

## Architectural Boundaries

- Runtime owns schedule definitions, recurrence evaluation, occurrence lifecycle, dispatch, replay, and quota enforcement
- Experience is a separate deployable composed of a Go BFF and compiled React/TypeScript UI
- Experience consumes Runtime only through governed published contracts
- Runtime persistence, broker topology, infrastructure details, and secrets remain private to Runtime
- Runtime and Experience build, release, deploy, scale, and roll back independently
- Repository co-location is a collaboration boundary, not an authority boundary

## Architecture Lineage

Repository topology follows accepted ADR-GLB-009 in the Scnehaux architecture repository. Implementation must remain aligned with the governing PAD, SAD, STD, ADR, and TDD artifacts.

## Status

Repository scaffold only. Implementation starts after the corresponding TDD baseline is established.
