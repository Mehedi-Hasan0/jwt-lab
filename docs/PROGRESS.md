# Progress

**Current milestone:** M0 · Foundation and production baseline
**Current step:** M0.1a
**Branch:** m0-foundation
**Last updated:** 2026-11-20

> The agent updates this file at the end of every step or sub-step: tick it, move "Current step" forward, and add a line to Session Notes. Sub-steps are added under their step once the split is agreed.

## M0 · Foundation and production baseline

- [ ] M0.1 Nest monorepo scaffold (apps, libs, strict TS, ESLint, Prettier)
  - [ ] M0.1a Monorepo layout: apps and shared libs
  - [ ] M0.1b Strict TypeScript
  - [ ] M0.1c ESLint with type-aware rules, plus Prettier
  - [ ] M0.1d Secrets hygiene: .gitignore, Node pin, guard test
- [ ] M0.2 docker-compose: Postgres + Redis with healthchecks
- [ ] M0.3 libs/config: zod validation, fail fast
- [ ] M0.4 libs/logger: redaction + request IDs + test
- [ ] M0.5 Health endpoints + graceful shutdown
- [ ] M0.6 HTTP security baseline (helmet, CORS allowlist, body limit)
- [ ] M0.7 CI pipeline (lint, typecheck, tests, audit, gitleaks)
- [ ] M0.8 Docs scaffolding (threat model, ADR-0001)

## M1 · Identity and token issuance

- [ ] M1.1 User model + seed
- [ ] M1.2 🧠 Register/login with argon2id + timing equalisation
- [ ] M1.3 Key generation + ADR-0002 key storage
- [ ] M1.4 JWKS + openid-configuration endpoints
- [ ] M1.5 🧠 Access token issuance
- [ ] M1.6 💥 Decode and tamper experiment

## M2 · The verification guard

- [ ] M2.1 Failing tests for every checklist item
- [ ] M2.2 🧠 verifyAccessToken + JwtAuthGuard + @CurrentUser
- [ ] M2.3 🧠 Scopes guard, 401 vs 403
- [ ] M2.4 Protect GET /orders + WWW-Authenticate
- [ ] M2.5 💥 JWKS cache experiment (auth-service down, unknown kid)

## M3 · Attack lab

- [ ] M3.1 libs/vulnerable
- [ ] M3.2 Attack harness
- [ ] M3.3 🧠 Attacks
  - [ ] a. alg none
  - [ ] b. Algorithm confusion
  - [ ] c. jku injection
  - [ ] d. kid path traversal
  - [ ] e. Weak HS256 secret
  - [ ] f. Audience confusion
  - [ ] g. Expired token replay
  - [ ] h. Tampered payload
- [ ] M3.4 CI attack job
- [ ] M3.5 💥 Mutation drill

## M4 · Refresh tokens

- [ ] M4.1 RefreshToken model
- [ ] M4.2 🧠 Issue at login (hashed at rest)
- [ ] M4.3 🧠 Atomic rotation
- [ ] M4.4 🧠 Reuse detection + ADR-0003 grace window
- [ ] M4.5 Logout + logout-everywhere
- [ ] M4.6 💥 Theft simulation

## M5 · Revocation and key lifecycle

- [ ] M5.1 🧠 jti denylist
- [ ] M5.2 🧠 Token versions
- [ ] M5.3 💥 Stale claims + live permission check
- [ ] M5.4 Key rotation script + runbook
- [ ] M5.5 💥 Emergency key removal
- [ ] M5.6 ADR-0004 Redis outage policy + tests

## M6 · Browser clients and the BFF

- [ ] M6.0 Setup: /etc/hosts entries + mkcert
- [ ] M6.1 Web app with storage modes
- [ ] M6.2 💥 XSS experiment in each mode
- [ ] M6.3 🧠 memory-cookie mode
- [ ] M6.4 🧠 BFF
- [ ] M6.5 💥 CSRF experiment + defences
- [ ] M6.6 Fix XSS + strict CSP

## M7 · Service-to-service

- [ ] M7.1 inventory-api
- [ ] M7.2 💥 Forwarded token rejected
- [ ] M7.3 🧠 Token exchange
- [ ] M7.4 🧠 Client credentials
- [ ] M7.5 💥 Zero-trust check
- [ ] M7.6 Client secret storage + ADR

## M8 · Payload size, sessions, and the hybrid

- [ ] M8.1 💥 Token size experiment
- [ ] M8.2 Session-based variant
- [ ] M8.3 💥 Benchmarks
- [ ] M8.4 🧠 Phantom token pattern
- [ ] M8.5 ADR-0005 sessions vs JWT

## M9 · Production hardening and capstone

- [ ] M9.1 STRIDE threat model review
- [ ] M9.2 Stryker mutation testing
- [ ] M9.3 @nestjs/jwt comparison
- [ ] M9.4 Hardened containers
- [ ] M9.5 Observability and alerts plan
- [ ] M9.6 Production readiness checklist
- [ ] M9.7 🧠 Capstone

## Decisions (ADRs)

| ADR  | Title      | Status  |
| ---- | ---------- | ------- |
| 0001 | Tech stack | Pending |

## Parked questions

Open decisions from the initial docs review. Decide each one at the step named.

- **M0.4** · Path-based redaction can't catch a JWT inside a log message or error. Add a second layer that scrubs JWT-shaped strings, or accept and document the gap?
- **M0.5** · `/health/ready` must reach the DB before Prisma arrives in M1.1. Bring Prisma in early, or use plain `pg`?
- **M1.5** · Setting `typ: at+jwt` declares the RFC 9068 profile, which makes `client_id` a required claim. Add it (and decide what the client is), or record the deviation?
- **M3.3f** · inventory-api doesn't exist until M7.1. How do we mint an `aud: inventory-api` token for the audience-confusion attack: a second configured audience in auth-service, or a test-only signer?
- **M6.3** · A refresh cookie with `Path=/auth/refresh` isn't sent to logout, so logout can't find the family to revoke. Widen the path to `/auth`, or move logout under `/auth/refresh`?

## Session notes

- 2026-11-19 · Project set up. Next: M0.1.
- 2026-11-20 · Reviewed the docs: fixed clear-cut inconsistencies and parked open decisions above. Learning loop updated: sub-steps, plan together, `build` gate, predict before Break. M0.1 split into M0.1a–d. Git repo initialised; remote `origin` is `github.com/Mehedi-Hasan0/jwt-lab`, so the M0.7 CI workflow can run there. Next: M0.1a.
