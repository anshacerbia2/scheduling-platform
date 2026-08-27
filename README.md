# Scheduling Platform

A reusable platform repository containing independently deployable Runtime and Experience applications.

## Repository Topology

```text
scheduling-platform/
├─ apps/
│  ├─ runtime/                  # SAD-013
│  │  └─ docs/
│  │     └─ 02-designs/
│  └─ experience/               # SAD-014
│     ├─ server/
│     ├─ web/
│     └─ docs/
│        └─ 02-designs/
├─ packages/
│  └─ contracts/                # governed published contracts / generated clients when justified
└─ deploy/
   ├─ runtime/                   # deployable-owned delivery manifests/configuration
   └─ experience/
```

## Architectural Boundaries

- Runtime owns schedule definitions, recurrence evaluation, occurrence lifecycle, dispatch, replay, and quota enforcement
- Experience is a separate deployable composed of a Go BFF and compiled React/TypeScript UI
- Experience consumes Runtime only through governed published contracts
- `packages/contracts` contains published cross-System contracts only; Runtime internals must not leak through it
- Runtime persistence, broker topology, infrastructure details, and secrets remain private to Runtime
- `deploy` contains deployable-owned delivery artifacts, not shared infrastructure authority
- Runtime and Experience build, version, release, deploy, scale, and roll back independently
- Repository co-location is a collaboration boundary, not an authority boundary
- No deployable may import another deployable's internal packages

## Architecture Lineage

Repository topology follows accepted ADR-GLB-009 in the Scnehaux architecture repository.

Implementation must remain aligned with the governing PAD, SAD, STD, ADR, and TDD artifacts.

## Status

Repository topology is established. Runtime and Experience implementation starts only after their respective TDD baselines are approved.
