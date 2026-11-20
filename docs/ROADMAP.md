# Roadmap

Milestones run in order. Each step follows the learning loop in `CLAUDE.md`.

A step that holds more than one idea is split into sub-steps (M0.1a, M0.1b, …), each teaching one idea. The split is agreed at the start of the step and tracked in `PROGRESS.md`.

Markers: 🔨 agent builds · 🧠 learner writes · 💥 break experiment

---

## M0 · Foundation and production baseline

**Goal:** a repo that is safe to build on before any JWT code exists.

**Steps**

1. 🔨 Create a Nest CLI monorepo:
   - apps: `auth-service`, `orders-api`
   - libs: `config`, `logger`, plus an empty `auth-verify`
   - TypeScript strict, ESLint, Prettier
2. 🔨 Write `docker-compose.yml` with Postgres and Redis, including healthchecks and named volumes.
3. 🔨 Build `libs/config`: zod-validated env with typed access, and fail fast on invalid config.
   - 💥 Start a service with a required variable missing, and show the startup failure.
4. 🔨 Build `libs/logger`: nestjs-pino with request IDs and redaction paths for:
   - headers: `authorization`, `cookie`, `set-cookie`
   - body fields: `password`, `refreshToken`, `accessToken`
   - the snake_case OAuth fields used from M7: `refresh_token`, `access_token`, `subject_token`, `client_secret`
   - 💥 Send a request with an `Authorization` header and prove the value is redacted. Add a test for it.
5. 🔨 Add health endpoints: `/health/live`, and `/health/ready` (checks DB and Redis). Add graceful shutdown hooks.
6. 🔨 Set up the HTTP security baseline: helmet, a CORS allowlist from config, a JSON body size limit, and `x-powered-by` disabled.
7. 🔨 Add CI: `npm ci`, lint, typecheck, unit tests, `npm audit --audit-level=high`, and gitleaks secret scanning.
8. 🔨 Scaffold the remaining docs: `threat-model.md` (assets, actors, trust boundaries) and `adr/0001-tech-stack.md`. `PROGRESS.md` and `learning-log.md` already exist; keep them updated.

**Production layer:** fail-fast config, log redaction, health checks, CI gates, secret scanning.

**Done when:** both apps start against Docker services, all tests pass, the redaction test passes, and the CI workflow is committed.

**Reflect:** Why fail fast on bad config? What could leak through logs in an auth system?

---

## M1 · Identity and token issuance (Modules 1–3)

**Goal:** issue correctly formed ES256 access tokens and publish the public keys.

**Steps**

1. 🔨 Write the Prisma `User` model (`id` uuid, `email` unique, `passwordHash`, `tokenVersion` int default 0, timestamps), plus a seed script.
2. 🧠 Build register and login with argon2id.
   - Unknown email and wrong password must return the same generic error.
   - Equalise timing by verifying against a dummy hash when the user doesn't exist.
   - 💥 Measure the response-time difference with and without the dummy hash.
3. 🔨 Add `npm run keys:generate`. It writes an ES256 key pair to `.keys/`. The `kid` is the date plus a short random suffix (for example `2026-11-20-a1b2`), so two keys made on the same day never collide. The service loads the private key from a path given in config.
   - Write `adr/0002-key-storage.md`: dev files vs a production secrets manager, KMS, or HSM.
4. 🔨 Expose public keys only:
   - `GET /.well-known/jwks.json` with sensible `Cache-Control`.
   - A minimal `/.well-known/openid-configuration` with `issuer` and `jwks_uri`.
5. 🧠 Build token issuance. The access token has:
   - claims: `iss`, `sub` (user id), `aud`, `iat`, `exp` (TTL from config, default 10 min), `jti` (uuid), `scope`
   - header: `typ: at+jwt` and `kid`
6. 💥 Decode an issued token with no key and read every claim. Then flip one character and show that verification fails, using a scratch script with `jose`.

**Tests:**

- The JWKS contains no private key material (no `d` field).
- All required claims are present.
- `exp − iat` equals the configured TTL.
- The payload contains no email or other personal data.

**Production layer:**

- Throttle `/auth/login`, keyed by IP + email.
- Emit audit events `login.success` and `login.failure` with no secrets.

**Reflect:** Why ES256 rather than HS256 here? What can an attacker do with your JWKS? Why equalise login timing?

---

## M2 · The verification guard (Module 6)

**Goal:** one reusable guard that enforces the full verification checklist.

**Steps**

1. 🔨 The agent writes **failing tests first**, one per checklist item:
   - malformed token
   - disallowed alg
   - unknown kid
   - bad signature
   - expired (using the injected Clock)
   - `nbf` in the future
   - wrong `iss`, wrong `aud`, wrong `typ`
   - missing Bearer scheme
   - token supplied in a query string (must be ignored)
2. 🧠 The learner implements `libs/auth-verify` to make them pass:
   - `verifyAccessToken()` using `createRemoteJWKSet`, with algorithms, issuer, audience, typ and clock tolerance all coming from config
   - `JwtAuthGuard`
   - verified claims attached to `req.auth`
   - a `@CurrentUser()` decorator
3. 🧠 Separate authorization from authentication: add a `@RequireScopes('orders:read')` guard. Use 401 for authentication failures and 403 for authorization failures.
4. 🔨 Protect `GET /orders` in orders-api. Return a `WWW-Authenticate: Bearer` header on 401 (RFC 6750).
5. 💥 Stop auth-service and show that orders-api keeps verifying from its cached JWKS. That's "the missing arrow". Then present an unknown `kid` and observe the refresh and its cooldown.

**Tests:** 100% branch coverage on `libs/auth-verify`.

**Production layer:** count auth failures by reason in logs or metrics, never in responses.

**Reflect:** Where exactly does authentication end and authorization begin in your code? What happens if the JWKS endpoint is down for an hour?

---

## M3 · Attack lab (Module 7)

**Goal:** see every classic attack succeed against a naive verifier and fail against yours.

**Steps**

1. 🔨 Build `libs/vulnerable`, a hand-rolled verifier using `node:crypto` that:
   - trusts `alg` from the header and accepts `none`
   - uses one key variable for any algorithm (give it an RS256 key pair, to reproduce the classic confusion attack)
   - honours `jku`
   - builds a file path from `kid`
2. 🔨 Build the attack harness: `npm run attacks -- --target=vulnerable|real` runs every attack and prints a table (attack, target, accepted or rejected).
3. 🧠 The learner writes each attack. The agent gives the recipe only on `hint`.
   - a. `alg: none`
   - b. algorithm confusion: HS256 signed with the RSA public key
   - c. `jku` injection, using an attacker JWKS server on `127.0.0.1:6666`
   - d. `kid` path traversal to a file whose contents are known, such as `/dev/null`
   - e. weak HS256 secret: sign with a weak secret, then crack it with a small wordlist script
   - f. audience confusion: replay an inventory-api token at orders-api
   - g. expired token replay
   - h. tampered payload
4. 🔨 Add a CI job that runs the suite against `real`. Any accepted attack fails the build.
5. 💥 Mutation drill. On a throwaway branch, remove one check at a time from `libs/auth-verify`. Record which test or attack catches each removal in `learning-log.md`.

**Production layer:** an ESLint rule plus a test that fail if anything outside `attacks/` imports `libs/vulnerable`.

**Done when:** every attack is accepted by `vulnerable` and rejected by `real`, and the mutation drill is recorded.

**Reflect:** For each attack, name the exact line of your guard that stops it.

---

## M4 · Refresh tokens (Module 4)

**Goal:** opaque, rotated refresh tokens with reuse detection that holds up under concurrency.

**Steps**

1. 🔨 Write the Prisma `RefreshToken` model:
   - `id`, `familyId`, `userId`, `parentId`
   - `tokenHash` (unique)
   - `usedAt`, `revokedAt`, `expiresAt`, `createdAt`
   - metadata: user agent, IP
2. 🧠 Issue a refresh token at login: 32 random bytes encoded as base64url. Store only the SHA-256 hash. Return both tokens.
3. 🧠 Build `POST /auth/refresh` with atomic rotation. Mark a token used with a conditional update, `updateMany({ where: { id, usedAt: null } })`, inside a transaction. Only one concurrent request may win.
4. 🧠 Add reuse detection:
   - When a used token is presented again, revoke the whole family, emit a `refresh.reuse_detected` audit event at warn level, and return 401.
   - Decide the grace-window behaviour for concurrent tabs in `adr/0003-refresh-grace-window.md`, then implement it.
5. 🔨 Add logout (revokes the current family) and logout-everywhere (revokes all of the user's families).
6. 💥 Theft simulation: a script where a legitimate client and an "attacker" race with the same refresh token. Show the family is revoked. Then show two tabs refreshing within the grace window both succeed.

**Tests:**

- Rotation works.
- Reuse revokes the family.
- Parallel refreshes outside the grace window produce exactly one success.
- Expired refresh tokens are rejected.
- The database never stores a plaintext token.

**Production layer:**

- Rate limit the refresh endpoint.
- Give refresh tokens an absolute lifetime and an idle timeout.
- Add a cleanup job for expired rows.
- Never log refresh tokens.

**Reflect:** Why hash refresh tokens but not access tokens? What breaks under concurrent refresh without the conditional update?

---

## M5 · Revocation and key lifecycle (Module 8)

**Goal:** revoke on demand, handle stale claims, and rotate keys without downtime.

**Steps**

1. 🧠 Build a `jti` denylist in Redis, using `SET` with an expiry equal to the token's remaining lifetime. Check it in the guard after the signature check. Add an admin endpoint to revoke a token.
2. 🧠 Add token versions:
   - a `ver` claim in each token
   - the user's current version cached in Redis, falling back to the DB
   - bump the version on password change and on logout-everywhere
3. 💥 Stale claims: demote an admin and show their token still says admin. Then add a live role check on `DELETE /orders/:id`.
4. 🔨 Key rotation: write a script and `docs/runbooks/key-rotation.md`.
   - Publish the new key first.
   - Switch signing after a configured delay.
   - Remove the old key after the maximum token lifetime.
5. 💥 Emergency key removal: delete the compromised key from the JWKS. Measure how long, with your cache settings, until every verifier rejects its tokens.
6. 🔨 Decide fail-closed vs fail-open for when Redis is down, in `adr/0004-revocation-store-outage.md`. Implement the decision and test it.

**Production layer:** audited revocations, a tested rotation runbook, and explicit behaviour when dependencies fail.

**Reflect:** Which strategy fits which case, and why? What is this system's revocation SLA in minutes?

---

## M6 · Browser clients and the BFF (Module 5)

**Setup:**

- Add `app.test` and `evil.test` to `/etc/hosts`. Different ports on localhost count as the _same site_, so cross-site experiments need different hostnames.
- Use mkcert for local HTTPS, so `Secure` cookies work.

**Steps**

1. 🔨 Build `apps/web` (Vite + TypeScript) with a mode switch: `localstorage`, `memory-cookie`, `bff`.
2. 💥 Plant an XSS bug: a notes field rendered with `innerHTML`. In each mode, run a payload that tries to read tokens and send them to `127.0.0.1:6666`. Record what it gets in each mode.
3. 🧠 Implement the `memory-cookie` mode:
   - access token in memory
   - refresh cookie: `__Secure-refresh`, `HttpOnly; Secure; SameSite=Strict; Path=/auth/refresh`
   - Note: the `__Host-` prefix requires `Path=/`, so it can't be used for a path-scoped cookie.
4. 🧠 Implement the BFF:
   - Tokens are stored server-side in Redis.
   - The browser gets `__Host-session` (`HttpOnly; Secure; SameSite=Lax; Path=/`).
   - The BFF proxies `/api/*` to orders-api, attaching the access token and refreshing it transparently.
5. 💥 Attempt CSRF from `evil.test` against the BFF. Compare `SameSite=Lax` with `SameSite=None`. Then add defences: an Origin / `Sec-Fetch-Site` check, plus a CSRF token or a required custom header.
6. 🔨 Fix the XSS properly (use `textContent`) and add a strict CSP. Show that the CSP blocks the payload even if the bug is reintroduced.

**Tests (Playwright):**

- In `bff` mode, `document.cookie`, `localStorage` and `sessionStorage` contain no token.
- Cross-site state-changing requests from `evil.test` are rejected.

**Production layer:** CSP, cookie prefixes, CSRF defence, and a CORS allowlist with credentials.

**Reflect:** What can XSS still do in BFF mode? Why does XSS defeat almost every storage choice?

---

## M7 · Service-to-service (Module 10)

**Goal:** every hop carries a token scoped to that hop alone.

**Steps**

1. 🔨 Build `inventory-api` with its own audience and scopes.
2. 💥 Have orders forward the user's token unchanged to inventory, and show it's rejected (wrong `aud`).
3. 🧠 Build a token exchange endpoint in auth-service (RFC 8693 shape):
   - `grant_type=urn:ietf:params:oauth:grant-type:token-exchange`
   - `subject_token`, `audience`, `scope`
   - It issues a token lasting 60–120 seconds, with `aud: inventory-api` and an `act` claim identifying orders.
   - Orders authenticates as a client when calling it.
4. 🧠 Implement the client credentials grant for a background worker. The service's token uses its client id as `sub`.
5. 💥 Zero-trust check: call inventory from "inside the network" with no token, and show it's rejected.
6. 🔨 Store client secrets hashed and support rotating them. Consider `private_key_jwt` in an ADR.

**Production layer:** least-privilege scopes per hop, short lifetimes, and audit logs for the `act` chain.

**Reflect:** Draw the token chain for one request. What does inventory know about who is calling, and on whose behalf?

---

## M8 · Payload size, sessions, and the hybrid (Modules 9, 11)

**Goal:** measure the trade-offs instead of assuming them.

**Steps**

1. 💥 Token size experiment: add N permission claims and record the token size.
   - Find the point where Node returns 431 (check the current default with `node --help`, under `--max-http-header-size`).
   - Find the point where a cookie exceeds 4 KB.
2. 🔨 Build a session-based variant of orders-api (express-session with a Redis store) for the same endpoints.
3. 💥 Benchmark with autocannon three ways: JWT local verify, session lookup in Redis, and JWT plus denylist check. Record p50, p99 and requests per second in `docs/benchmarks.md`.
4. 🧠 Implement the phantom token pattern:
   - The BFF holds an opaque session.
   - It exchanges the session for a short-lived internal JWT, cached until near expiry.
   - Internal services accept only the internal audience.
5. 🔨 Write `adr/0005-sessions-vs-jwt.md`, using your benchmark numbers.

**Reflect:** Given your numbers, where are JWTs worth it in this system, and where aren't they?

---

## M9 · Production hardening and capstone (Module 12)

**Goal:** prove the system is production-grade, and prove the learner can explain it.

**Steps**

1. 🔨 Review the threat model using STRIDE for each trust boundary. Every threat must link to a test or to a documented, accepted risk.
2. 🔨 Run mutation testing with Stryker on `libs/auth-verify`. Explain every surviving mutant.
3. 🔨 Library comparison: rebuild verification with `@nestjs/jwt`. List every option you had to set to match your guard's guarantees.
4. 🔨 Containers: multi-stage Dockerfiles, a non-root user, and no keys baked into images. Keys are mounted at runtime.
5. 🔨 Write an observability plan listing the alerts:
   - spikes in 401s by reason
   - `refresh.reuse_detected` events
   - JWKS fetch failures
   - login failure rate
6. 🔨 Complete the production readiness checklist below, linking evidence for each item.
7. 🧠 Capstone:
   - The learner draws the architecture from memory.
   - The learner writes a one-minute interview answer in `learning-log.md`.
   - The learner walks through login → refresh → exchange → revoke.
   - The agent plays interviewer and asks follow-ups.

## Production readiness checklist

- [ ] Config validated at startup; no secrets in the repo, images, or logs
- [ ] Private keys kept outside images; rotation runbook tested
- [ ] Algorithm allowlist; `iss`, `aud`, `exp`, `nbf` and `typ` checked; JWKS cached with a refresh cooldown
- [ ] Access token TTL of 15 minutes or less
- [ ] Refresh tokens rotated and hashed, with family revocation and both absolute and idle expiry
- [ ] Rate limits on login, refresh, and token exchange
- [ ] Generic client errors; detailed, redacted server logs; audit events for every auth action
- [ ] Cookies HttpOnly, Secure, SameSite and prefixed; CSRF defence on cookie-authenticated endpoints; strict CSP
- [ ] Attack suite and mutation tests running in CI
- [ ] Behaviour during Redis or DB outages decided in an ADR and tested
- [ ] Dependency audit and secret scanning running in CI
- [ ] Containers running as non-root on minimal base images
