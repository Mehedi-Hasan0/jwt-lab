# Progress

**Current milestone:** M0 · Foundation and production baseline
**Current step:** M0.1
**Branch:** m0-foundation
**Last updated:** 2026-11-19

> The agent updates this file at the end of every step: tick the step, move "Current step" forward, and add a line to Session Notes.

## M0 · Foundation and production baseline

- [ ] M0.1 Nest monorepo scaffold (apps, libs, strict TS, ESLint, Prettier)
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

- (Questions to come back to later)

## Session notes

- 2026-11-19 · Project set up. Next: M0.1.
