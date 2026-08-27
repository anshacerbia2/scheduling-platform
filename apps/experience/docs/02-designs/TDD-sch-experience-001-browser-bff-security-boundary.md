---
doc_meta:
  id: TDD-sch-experience-001
  title: Scheduling Experience Browser and BFF Security Boundary
  owner: Scheduling Platform Team
  version: 1.0.0
  status: approved
  classification: restricted
  parent_sad: SAD-014
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-27
---
# Scheduling Experience Browser and BFF Security Boundary

## Purpose

Define the same-origin Go BFF security boundary for Scheduling Experience, including OIDC login, opaque application sessions, delegated token custody, CSRF, Origin/Host enforcement, context binding, step-up, redirect policy, and telemetry privacy.

## Scope

Covers browser/BFF security and server-to-server Scheduling API mediation. It excludes Schedule business semantics and React screen composition.

## Technical Context

Browser JavaScript communicates only with the Scheduling Experience origin. It never receives long-lived access/refresh tokens or infrastructure credentials and never reaches Scheduling PostgreSQL, RabbitMQ, Kafka, or direct target endpoints.

The BFF owns an opaque application session. Session backing storage is a replaceable enterprise web-runtime concern; it is not Scheduling authority and must be HA for the declared Experience availability profile.

## Component Design

```text
Browser
  -> Go BFF
      -> Session Manager
      -> OIDC Adapter
      -> Context Guard
      -> CSRF/Origin Guard
      -> Scheduling API Adapter
```

Cookie name: `__Host-scnehaux_sched_session`. It is Secure, HttpOnly, Path=/, has no Domain attribute, and uses SameSite=Lax unless an approved browser flow requires stricter behavior.

The browser receives a per-session CSRF token through a same-origin bootstrap response and sends it in `X-CSRF-Token` for state-changing requests.

## Data Model

Session state contains opaque session ID, Principal ID, delegated token material or token handle held server-side, assurance level, current application/Tenant context, creation/last-use/expiry, CSRF secret hash, and token-expiry metadata.

Context switch increments `context_generation`. Every pending mutation and query cache key is bound to that generation.

## API / Interface

BFF routes:

- `GET /auth/login`
- `GET /auth/callback`
- `POST /auth/logout`
- `GET /api/session`
- `POST /api/context`
- `/api/scheduling/*` mediated routes

State-changing routes require valid session, CSRF token, allowed Origin, allowed Host, and server-side authorization context. Privileged replay/quota override routes require a current step-up assurance marker.

## Algorithms / Logic

Login uses Authorization Code with PKCE and state/nonce validation. Session ID rotates after login, step-up, privilege elevation, and context switch.

Context switch:

1. verify requested context through governed server-side context source
2. update server-side session
3. increment `context_generation`
4. return new generation
5. browser cancels in-flight requests, clears query/mutation/form state, then activates new context

BFF rejects any request carrying a stale generation.

## Configuration

- absolute session lifetime 8 h
- idle timeout 30 min
- step-up validity 10 min for privileged operations
- allowed origins exact-list
- redirect allowlist exact path/host policy
- request body limit 1 MiB
- upstream timeout 10 s for interactive operations

Production settings can tighten these bounds.

## Security Notes

CSP denies object/embed, restricts script/style/connect/frame sources to governed origins, and uses frame-ancestors protection. HSTS is enabled at ingress. No wildcard authenticated CORS exists.

Tokens, session IDs, CSRF material, trigger payloads, and authorization headers are redacted from logs/traces.

## Failure Handling

Expired/invalid session returns 401 and clears the cookie. Context mismatch returns 409 and forces client state reset. Upstream Scheduling timeout returns a safe 502/504 problem without replaying non-idempotent mutations unless the original idempotency identity is retained.

## Observability

Metrics cover login outcome, session validation, CSRF/origin rejection, context-switch count, upstream latency/error, and step-up challenge. Traces correlate browser request -> BFF -> Scheduling API without credentials.

## Performance Notes

Static React assets are immutable/cacheable. BFF does no large aggregation; list/history pagination remains server-side. Session lookup latency must remain a small fraction of the 500 ms interactive p95 target.

## Testing Strategy

Blocking security/E2E tests cover PKCE/state/nonce, cookie flags, fixation/rotation, CSRF, Origin/Host spoofing, context stale-tab behavior, step-up expiry, redirect abuse, token leakage scanning, no direct broker/DB access, and authorization-negative paths.

## Operational Notes

Experience outage does not stop Scheduling Runtime. Session-store degradation fails privileged mutations closed. Rollback is independent from Runtime and does not require Schedule migration.

## Traceability

Implements SAD-014 browser-facing authentication/session, CSRF, context, and API-mediation constraints. Scheduling authority remains in SAD-013 and its runtime TDDs.
