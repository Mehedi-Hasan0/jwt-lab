> 📘 A self-study guide to JSON Web Tokens for backend engineers. Modules 1–3 cover the topics in the free preview of the source article, explained in my own words. Modules 4–12 reconstruct what the paid section most likely covers. They are not the author's text.

**Source:** [JWT Authentication Explained: How It Works, Where It Fails, and When to Use It](https://designgurus.substack.com/p/jwt-in-90-minutes-a-backend-engineers) · Arslan Ahmad · _System Design Nuggets_ (Design Gurus) · Oct 28, 2026 · Paid post; the free preview covers its Parts 1–3

**Legend**

- 📖 Topic from the source's free preview, explained in my own words
- 🧩 Reconstructed: what the paywalled section most likely covers
- 💡 Key idea · ⚠️ Common mistake · ✅ Check yourself (answers in the Answer Key)

---

# 🧭 Start Here

## How to use this guide

1. Do one or two modules per sitting. Each takes 10–20 minutes.
2. Answer the ✅ questions before opening the Answer Key.
3. Do the Hands-On Lab after Module 6. The attacks in Module 7 stick better once you've watched a forged token get rejected.
4. Come back to the Cheat Sheet before interviews, code reviews, or any auth design work.

## Study plan

| Session | Modules      | Outcome                                                          |
| ------- | ------------ | ---------------------------------------------------------------- |
| 1       | 1–3          | Know what a token contains and how it is signed                  |
| 2       | 4–5          | Understand login, refresh, and where tokens live                 |
| 3       | 6–7 + Lab    | Verify tokens correctly and recognise the attacks                |
| 4       | 8–10         | Handle revocation, payload design, and service-to-service calls  |
| 5       | 11–12 + Quiz | Choose between JWTs and sessions, and explain it in an interview |

## Progress

- [ ] Module 1 · What a JWT is and why it exists
- [ ] Module 2 · Anatomy of a token
- [ ] Module 3 · Signing algorithms and key management
- [ ] Module 4 · The authentication flow
- [ ] Module 5 · Where the client keeps the token
- [ ] Module 6 · Verifying a token correctly
- [ ] Hands-On Lab
- [ ] Module 7 · Common vulnerabilities
- [ ] Module 8 · Lifecycle and revocation
- [ ] Module 9 · What belongs in the payload
- [ ] Module 10 · JWTs between services
- [ ] Module 11 · When sessions are the better choice
- [ ] Module 12 · JWTs in a system design interview
- [ ] Scenario Quiz

---

# 📖 Module 1 · What a JWT Is and Why It Exists

🎯 **You will be able to:** explain the problem JWTs solve, and why a signed token is not a secret.

## The question behind every request

A protected API has to answer two things on every request: who is calling, and are they allowed to do this.

## The session approach

At login, the server stores a record about the user and gives the client a random ID, usually in a cookie.

On each request, the server looks up that ID to find the user.

It's simple and reliable. The cost appears at scale. Every service and every instance needs access to the same session store, and every request pays for a lookup.

## The token approach

A JWT turns this around.

Instead of an ID that points to data held on the server, the client receives the data itself, signed by the issuer.

Any service holding the right key can check the signature and trust the contents. There is no call to the issuer and no shared store.

> 💡 A session ID is a reference to data on the server. A JWT is a signed statement the client carries.

## Definition

A JWT (RFC 7519, pronounced "jot") is a compact, URL-safe string that carries **claims**: statements about a subject, usually a user. The claims are written as JSON and protected by a digital signature or a MAC.

Two properties make it useful:

- **Self-contained.** What you need to authorize the request is inside the token.
- **Verifiable without the issuer.** A service needs a key, not a network call.

## Signed is not encrypted

A standard JWT is **encoded** and **signed**. Anyone can reverse the encoding (base64url), and any change to the token is detectable. But nothing in it is hidden: whoever holds it can read every claim.

| Form            | Spec     | Protects                        | Typical use                                 |
| --------------- | -------- | ------------------------------- | ------------------------------------------- |
| JWS (signed)    | RFC 7515 | Integrity: detects tampering    | What people almost always mean by "JWT"     |
| JWE (encrypted) | RFC 7516 | Confidentiality, plus integrity | When the claims themselves must stay hidden |

> ⚠️ Never put passwords, secrets, or sensitive personal data in a signed JWT.

## Where JWTs go wrong

Most JWT problems come from four decisions:

- using JWTs where sessions would be simpler,
- storing them where scripts can read them,
- giving them lifetimes too long to revoke,
- verifying them carelessly.

Modules 5–8 take these one at a time.

✅ **Check yourself**

- **Q1.** What does a session-based system keep on the server that a JWT-based system avoids?
- **Q2.** Can a user read the claims inside their own JWT? Why?

---

# 📖 Module 2 · Anatomy of a Token

🎯 **You will be able to:** read any JWT and say what each part does.

A JWT is three base64url strings joined by dots:

```text
header.payload.signature
eyJhbGciOi... . eyJzdWIiOi... . SflKxwRJSM...
```

**base64url** is base64 with URL-safe characters (`-` and `_` instead of `+` and `/`) and no padding. It is an encoding, not encryption.

## The header

```json
{
  "alg": "ES256",
  "typ": "JWT",
  "kid": "2026-11"
}
```

- **alg:** the signing algorithm.
- **typ:** the token type.
- **kid:** the key ID. It tells the verifier which key signed the token, which is essential once keys rotate.

> ⚠️ The header is written by whoever made the token. Until the signature is verified, treat it as untrusted input (see Module 7).

## The payload: registered claims

| Claim | Name       | Meaning                                   | What the verifier does                      |
| ----- | ---------- | ----------------------------------------- | ------------------------------------------- |
| iss   | Issuer     | Who created the token                     | Must equal the expected issuer              |
| sub   | Subject    | Who the token is about (a stable user ID) | Treats it as the caller's identity          |
| aud   | Audience   | Intended recipient service(s)             | Must include this service                   |
| exp   | Expiration | Time after which it's invalid             | Rejects it if in the past                   |
| nbf   | Not before | Time before which it's invalid            | Rejects it if not yet reached               |
| iat   | Issued at  | When it was created                       | Uses it for "valid after" checks            |
| jti   | JWT ID     | Unique ID for this token                  | Uses it for revocation and replay detection |

All times are **NumericDate**: seconds since the Unix epoch.

## The payload: custom claims

You add your own claims for roles, scopes, tenant IDs, and so on.

```json
{
  "iss": "https://auth.example.com",
  "sub": "user_8f3a2c",
  "aud": "orders-api",
  "iat": 1795600200,
  "exp": 1795600800,
  "jti": "b7e1c9d4-5f2a-4e8b-9c31-0a6d2f7e4b19",
  "tenant_id": "acme",
  "scope": "orders:read orders:write"
}
```

Here `exp − iat = 600`, so this token lives for 10 minutes.

Keep the payload small, because it travels with every request. Keep it non-sensitive, because anyone can read it.

## The signature

```text
signature = Sign( base64url(header) + "." + base64url(payload), key )
```

The signature covers both the header and the payload. Change a single character in either, and it no longer matches.

A valid signature proves two things:

1. The token was made by someone holding the signing key.
2. It hasn't changed since.

> 💡 The signature proves origin and integrity. It does NOT prove three things: that the token is still appropriate, that it was meant for your service, or that the holder is the person it was issued to.

## Try it: decode a token locally

```js
// Node.js 16+: decode only. This does NOT verify anything.
const [header, payload] = token.split(".");
console.log(JSON.parse(Buffer.from(header, "base64url").toString()));
console.log(JSON.parse(Buffer.from(payload, "base64url").toString()));
```

✅ **Check yourself**

- **Q3.** Which header field tells the verifier which key to use, and why does it matter during key rotation?
- **Q4.** What two things does a valid signature prove?

---

# 📖 Module 3 · Signing Algorithms and Key Management

🎯 **You will be able to:** choose between symmetric and asymmetric signing, and rotate keys safely.

## Symmetric signing: HS256

HS256 is HMAC with SHA-256. One shared secret both signs and verifies.

- ✔️ Simple and fast. Fine when a single application issues and verifies its own tokens.
- ✖️ Every service that can verify can also forge. One compromised service compromises authentication everywhere.
- ✖️ A weak secret can be brute-forced offline from a single captured token.

## Asymmetric signing: RS256, ES256, EdDSA

These use a key pair:

- The **private key** signs. It never leaves the auth service.
- The **public key** verifies. It can be shared with anyone.

A compromised verifier holds only a public key, so it cannot mint tokens.

| Algorithm | Type               | Signature size           | Notes                          |
| --------- | ------------------ | ------------------------ | ------------------------------ |
| HS256     | HMAC (symmetric)   | 32 bytes                 | Shared secret; single app only |
| RS256     | RSA                | 256 bytes (2048-bit key) | Most widely supported          |
| ES256     | ECDSA, P-256 curve | 64 bytes                 | Smaller tokens                 |
| EdDSA     | Ed25519            | 64 bytes                 | Modern; check library support  |

> 💡 If more than one party verifies tokens, use asymmetric signing.

## Publishing keys: JWKS

The issuer publishes its public keys as a **JSON Web Key Set (JWKS)** at a well-known URL. In OpenID Connect, the `jwks_uri` field of `/.well-known/openid-configuration` points to it.

```json
{
  "keys": [
    {
      "kty": "EC",
      "crv": "P-256",
      "kid": "2026-11",
      "use": "sig",
      "alg": "ES256",
      "x": "…",
      "y": "…"
    }
  ]
}
```

Verifiers handle the set like this:

1. Fetch it and cache it.
2. For each token, pick the key whose `kid` matches.
3. On an unknown `kid`, refresh the set, but rate-limit those refreshes. Otherwise attackers can flood the key endpoint with made-up `kid` values.

## Rotating keys safely

1. Generate a new key pair and add its public key to the JWKS, next to the old one.
2. Wait until verifier caches have picked up the new set.
3. Start signing new tokens with the new key and its new `kid`.
4. Keep the old public key published until every token it signed has expired.
5. Remove the old key.

✅ **Check yourself**

- **Q5.** Why is HS256 a poor fit for a system with 20 services that all verify tokens?
- **Q6.** Why must the old public key stay in the JWKS for a while after rotation?

---

# 🧩 Module 4 · The Authentication Flow

🎯 **You will be able to:** walk through login, authenticated requests, and refresh, including rotation and reuse detection.

## Basic flow

1. The user sends credentials to the auth service.
2. The auth service checks them and issues a signed access token.
3. The client sends that token on every request:

```http
Authorization: Bearer eyJhbGciOiJFUzI1NiIsImtpZCI6IjIwMjYtMTEifQ.eyJzdWIiOi...
```

4. The receiving service verifies the token on its own. It finds the key by `kid`, checks the signature, then checks expiry, issuer and audience.
5. Only then does it read the claims and decide what the caller may do.

```text
Client              Auth Service               Orders API
  | -- credentials --> |                           |
  | <-- signed JWT --- |                           |
  | -- GET /orders  (Bearer <JWT>) --------------> |
  |                    |       verify locally with cached public key
  | <-- 200 OK ----------------------------------- |
```

Notice there is no arrow from Orders API back to Auth Service. That missing arrow is the whole appeal of JWTs. It is also the source of most of the difficulty that follows.

> ⚠️ "Bearer" means whoever holds the token is treated as the user. Nothing checks that the presenter is the person it was issued to.

> 💡 Authentication answers "who is this?" Authorization answers "may they do this?" The token supplies input for the second question, but your service still has to ask it.

## Why one token isn't enough

A single token forces a bad trade-off:

- **Long-lived (a week):** a stolen token works for a week, and you can't recall it.
- **Short-lived (10 minutes):** the user has to log in every 10 minutes.

The standard fix is to split the job between two tokens.

## Access tokens and refresh tokens

|          | Access token                  | Refresh token                          |
| -------- | ----------------------------- | -------------------------------------- |
| Format   | JWT                           | Usually an opaque random string        |
| Lifetime | 5–15 minutes                  | Hours to weeks                         |
| Sent to  | Every API, on every request   | Only the auth service's token endpoint |
| Checked  | Statelessly, by many services | Statefully, by one service             |
| Purpose  | Authorize requests            | Get a new access token                 |

The refresh token always goes back to the issuer, so the issuer gets a chance to say no on every refresh. Maybe the account was disabled, the password changed, or the user logged out elsewhere. Then the refresh fails, and access ends within one access-token lifetime.

> 💡 Keep stateless checks on the hot path, where requests are frequent. Push the stateful check to the cold path: a few refreshes an hour.

## Refresh token rotation

Each time a refresh token is used, the server issues a new one and invalidates the old. Every refresh token works exactly once.

```text
Client                                    Auth Service
  | -- refresh token R1 -------------------> |  valid? unused? not revoked?
  | <-- new access token + refresh R2 ------ |  mark R1 as used
```

## Reuse detection

Suppose an attacker steals R1. Either the attacker or the real client will use it first. Whoever presents it second is presenting a spent token.

The server can't tell which one is the attacker, and it doesn't need to. Reuse of a spent token proves the token was copied.

So the server revokes the whole **token family**: every refresh token descended from that login. Both parties are logged out. The real user signs in again; the attacker can't.

> ⚠️ Two browser tabs refreshing at the same moment can trigger reuse detection by accident. Many implementations accept the previous token for a few seconds of grace.

✅ **Check yourself**

- **Q7.** Why does splitting into access and refresh tokens make revocation practical?
- **Q8.** In reuse detection, why is it fine that the server can't tell the attacker from the real user?

---

# 🧩 Module 5 · Where the Client Keeps the Token

🎯 **You will be able to:** pick a storage strategy for web and mobile clients and explain its trade-offs.

A token is a bearer credential. Where it lives matters as much as how it was signed.

## Options at a glance

| Option                          | Can XSS steal it?                             | CSRF exposure               | Survives reload | Verdict                       |
| ------------------------------- | --------------------------------------------- | --------------------------- | --------------- | ----------------------------- |
| localStorage / sessionStorage   | Yes                                           | None                        | Yes             | Avoid for tokens              |
| HttpOnly cookie                 | No; it can only use it while the page is open | Yes; mitigate with SameSite | Yes             | Good                          |
| In-memory variable              | Harder                                        | None                        | No              | Good for access tokens        |
| BFF (tokens stay on the server) | Nothing in the browser to steal               | Same as sessions            | Yes             | Strongest for web             |
| iOS Keychain / Android Keystore | Not applicable                                | Not applicable              | Yes             | Use for mobile refresh tokens |

## localStorage and sessionStorage

Any JavaScript on the page can read them. That includes injected scripts from a vulnerable dependency, a compromised analytics tag, or an unescaped comment.

One such script can take the token and use it from the attacker's own machine.

## Cookies

Three flags matter:

- **HttpOnly:** JavaScript can't read the cookie.
- **Secure:** the cookie is sent only over HTTPS.
- **SameSite=Lax or Strict:** the browser doesn't attach it to most cross-site requests.

The trade-off is CSRF (cross-site request forgery). Browsers attach cookies automatically, so another site can trigger requests that carry them. SameSite stops most of this, and CSRF tokens or Origin checks cover the rest.

Cookies also cap at about 4 KB.

## The common web pattern

1. The access token lives **in memory**.
2. The refresh token lives in an **HttpOnly, Secure, SameSite=Strict cookie**, scoped to the refresh endpoint's path.
3. On page load, the app calls the refresh endpoint. The cookie goes along, and a new access token comes back.

## Backend for Frontend (BFF)

A BFF is a thin server dedicated to one web app. It:

- performs the login,
- holds the access and refresh tokens on the server,
- gives the browser a plain session cookie,
- attaches the access token when forwarding each call.

Tokens never reach browser JavaScript. Yes, this is a session. Module 11 explains why that's no coincidence.

## Sender-constrained tokens

The options above try to prevent theft. This approach makes stolen tokens useless instead.

With **DPoP** (RFC 9449) or **mutual TLS** (RFC 8705), the token is bound to a key the client holds. Each request proves the caller has that key, so a copied token without the key is rejected.

It adds complexity, so it's mostly used for high-value APIs.

✅ **Check yourself**

- **Q9.** What can an XSS payload do with an HttpOnly cookie, and what can't it do?
- **Q10.** Why is a BFF the strongest option for browser apps?

---

# 🧩 Module 6 · Verifying a Token Correctly

🎯 **You will be able to:** write a verifier that rejects forged, expired, and misdirected tokens.

Most JWT vulnerabilities aren't broken cryptography. They're verification code that skips a step or trusts the wrong input.

## The verification checklist

Run these in order and reject the token at the first failure.

- [ ] **Parse strictly.** Exactly three parts, valid base64url, valid JSON.
- [ ] **Choose the algorithm yourself.** Configure an allowlist, for example only ES256, and reject anything else the header names.
- [ ] **Get the key from a trusted source.** Look up the `kid` in your own cached key set from your configured issuer. Never fetch a key from a URL found inside the token.
- [ ] **Verify the signature** with a maintained library.
- [ ] **Check time claims.** `exp` has not passed and `nbf` has been reached, with 30–60 seconds of leeway for clock skew.
- [ ] **Check the issuer.** `iss` matches exactly.
- [ ] **Check the audience.** `aud` includes this service. This is the check most often forgotten.
- [ ] **Check the token type.** For example, `typ` is `at+jwt` for access tokens (RFC 9068).
- [ ] **Only now, read the claims** and authorize.

> ⚠️ Never decode the payload and act on it before verifying. Until the signature checks out, every byte is attacker-controlled. That includes choosing a key based on a tenant ID found in the payload.

## Reference implementation (Node.js, `jose`)

```js
import { createRemoteJWKSet, jwtVerify } from "jose";

// Fetched once, cached, refreshed on unknown kid (with a cooldown).
const JWKS = createRemoteJWKSet(
  new URL("https://auth.example.com/.well-known/jwks.json"),
);

export async function authenticate(req) {
  const [scheme, token] = (req.headers.authorization ?? "").split(" ");
  if (scheme !== "Bearer" || !token) throw new Error("unauthorized");

  const { payload } = await jwtVerify(token, JWKS, {
    algorithms: ["ES256"], // we choose, not the header
    issuer: "https://auth.example.com",
    audience: "orders-api",
    typ: "at+jwt",
    clockTolerance: 30, // seconds of skew allowed
  });

  return payload; // now, and only now, trust the claims
}
```

> 💡 Put every check into one verify call with explicit expectations. Treat any "decode without verify" function as a debugging tool only.

✅ **Check yourself**

- **Q11.** Why must the verifier, not the token header, decide which algorithm is acceptable?
- **Q12.** Name the check most often forgotten, and the attack it prevents.

---

# 🧪 Hands-On Lab · Sign, Verify, and Break Tokens

**Setup:** run `npm install jose`, save the code below as `lab.mjs`, and run it with `node lab.mjs`.

```js
import { generateKeyPair, SignJWT, jwtVerify } from "jose";

const { publicKey, privateKey } = await generateKeyPair("ES256");

const token = await new SignJWT({ scope: "orders:read" })
  .setProtectedHeader({ alg: "ES256", typ: "at+jwt", kid: "lab-1" })
  .setIssuer("https://auth.example.com")
  .setSubject("user_123")
  .setAudience("orders-api")
  .setIssuedAt()
  .setExpirationTime("10m")
  .sign(privateKey);

console.log("token:", token);

const { payload } = await jwtVerify(token, publicKey, {
  algorithms: ["ES256"],
  issuer: "https://auth.example.com",
  audience: "orders-api",
  typ: "at+jwt",
});
console.log("verified:", payload);
```

## Exercises

- [ ] Decode the header and payload with the snippet from Module 2. Confirm you can read every claim without any key.
- [ ] Change `audience` in the verify call to `'billing-api'`. You should get a claim validation error.
- [ ] Change one character in the middle (payload) segment of the token. You should get a signature verification error.
- [ ] Set the expiration to `'2s'`, wait 3 seconds, then verify. You should get an expiry error.
- [ ] Forge an unsigned token and confirm it's rejected:

```js
const b64 = (o) => Buffer.from(JSON.stringify(o)).toString("base64url");
const forged =
  b64({ alg: "none", typ: "at+jwt" }) +
  "." +
  b64({
    sub: "admin",
    iss: "https://auth.example.com",
    aud: "orders-api",
    exp: Math.floor(Date.now() / 1000) + 600,
  }) +
  ".";

try {
  await jwtVerify(forged, publicKey, { algorithms: ["ES256"] });
  console.log("❌ accepted a forged token!");
} catch (e) {
  console.log("✅ rejected:", e.code);
}
```

- [ ] Turn each exercise into an automated test in a real project. These become your regression tests against the attacks in Module 7.

---

# 🧩 Module 7 · Common Vulnerabilities

🎯 **You will be able to:** recognise the classic JWT attacks and name the defence for each.

Almost every JWT attack exploits the same mistake: letting the token decide how it should be verified.

## Attack map

| Attack                  | What the attacker does                           | Root cause                              | Defence                                 |
| ----------------------- | ------------------------------------------------ | --------------------------------------- | --------------------------------------- |
| alg: none               | Strips the signature and sets alg to none        | Library obeys the header                | Algorithm allowlist                     |
| Algorithm confusion     | Signs an HS256 token using your RSA public key   | One key used for two algorithm families | Pin the algorithm; use typed keys       |
| jku / x5u / jwk headers | Points the verifier at the attacker's own key    | Keys taken from the token               | Ignore them, or use a strict allowlist  |
| kid injection           | Path traversal or SQL injection through kid      | Untrusted kid used in lookups           | Treat kid as a key into a fixed map     |
| Weak HMAC secret        | Brute-forces the secret offline                  | Short or guessable secret               | 256+ random bits in a secrets manager   |
| Audience confusion      | Replays a token for service A at service B       | Missing aud check                       | Always check aud; separate environments |
| Token leakage           | Harvests tokens from URLs, logs, online decoders | Tokens kept in the wrong places         | Headers or cookies only; redact logs    |
| Library bugs            | Exploits a flaw in the verifier itself           | Unpatched dependency                    | Patch quickly; keep forged-token tests  |

## alg: none

The JWT specs define an algorithm called `none`, meaning "unsigned."

The attack works like this:

1. Take a valid token and edit the payload to claim admin rights.
2. Set `alg` to `none`.
3. Delete the signature.

A library that obeys the header skips verification and accepts the forgery. Several widely used libraries behaved exactly this way when the issue was publicized in 2015.

## Algorithm confusion

The setup:

- The server issues RS256 tokens.
- Its code calls something like `verify(token, key)` with the RSA public key.
- It lets the header pick the algorithm.

The attack:

1. Forge a token with `alg: HS256`.
2. Compute an HMAC over it, using the server's public key as the secret. The public key is, by definition, public.

The server sees HS256 and uses the key it was given, the public key, as an HMAC secret. The HMAC matches, and the forgery passes.

## Header fields that point at keys

Three header fields can point at keys:

- `jku` points to a key set URL.
- `x5u` points to a certificate URL.
- `jwk` embeds a key directly.

If the verifier honors them, the attacker signs with their own private key and points the verifier at their own public key. Trusted keys must come from your configuration, never from the token.

## kid injection

`kid` is used to find the key before verification, so it is attacker-controlled at that moment.

If your code builds a file path from it, an attacker can point the lookup at a file whose contents they already know, such as `/dev/null`. They then sign with that known value. If your code puts `kid` into SQL, you have SQL injection.

## Weak HMAC secrets

An HS256 token contains everything needed to test a guess: the signed input and the resulting signature.

With tools like hashcat, an attacker can test billions of guesses per second offline. Secrets like "secret" or a company name fall in seconds. RFC 7518 requires at least 256 bits for HS256.

## Audience confusion

Two services trust the same issuer. A token issued for the low-privilege notifications service is replayed against billing.

If billing doesn't check `aud`, the token passes. The signature is valid, the issuer is correct, and it hasn't expired. Every check passed except the one that mattered.

The same applies across tenants and environments. Use separate keys for staging and production.

## Token leakage

Tokens get stolen without any cryptography:

- Tokens in URLs land in browser history, server and proxy logs, and Referer headers.
- A debug line that logs the full Authorization header ships tokens into your log pipeline.
- Tokens pasted into online decoders go to a third party.

## Library bugs

In 2022, CVE-2022-21449 ("psychic signatures") made Java 15 through 18 accept an all-zero ECDSA signature as valid for any message. Every ES256 verifier on those versions accepted forged tokens with a blank signature.

Use widely adopted libraries, patch them quickly, and keep the Lab's forged-token tests in CI.

✅ **Check yourself**

- **Q13.** In algorithm confusion, why does the attacker's forged HMAC verify?
- **Q14.** Why is `kid` dangerous even though it's "just an ID"?

---

# 🧩 Module 8 · Lifecycle and Revocation

🎯 **You will be able to:** choose a revocation strategy and explain what it costs.

With sessions, revocation is trivial: delete the row, and the next request fails.

With a stateless JWT, there is no row. The token is valid because of what it contains, and nothing you change on the server alters that. It is accepted until `exp`.

This is not a bug. It is the price of the property that made JWTs attractive. If verification needs no lookup, then revocation without a lookup is impossible.

> 💡 Every revocation strategy is a decision about how much state to bring back, and where.

## Strategies compared

| Strategy                     | Revocation delay                | Per-request cost       | Best for                          |
| ---------------------------- | ------------------------------- | ---------------------- | --------------------------------- |
| Short lifetimes + refresh    | Up to one access-token lifetime | None                   | Most applications                 |
| jti denylist                 | Immediate                       | Shared-store lookup    | Revoking specific tokens          |
| Per-user token version       | Immediate or near-immediate     | Cached per-user lookup | "Log out everywhere"              |
| Remove signing key from JWKS | After verifier caches refresh   | None                   | Key compromise; logs everyone out |

## Strategy 1: Short lifetimes

Accept the gap and make it small.

With 5-minute access tokens and refresh rotation, a disabled account loses access within 5 minutes. It costs nothing on the hot path.

The question for the business is concrete: how long can a fired employee or a compromised account keep acting? If a few minutes is acceptable, stop here.

## Strategy 2: A denylist by jti

On revocation:

1. Write the token's `jti` to a fast shared store such as Redis.
2. Set the entry's time-to-live to the token's remaining lifetime, so it cleans itself up.
3. Have every verifier check the list after verifying the signature.

> ⚠️ Be honest about what you've built. A shared-store call on every request is the session lookup JWTs were meant to remove. It can still be worth it, because a denylist is small and simple, but the system is no longer stateless.

## Strategy 3: Token versions

Revoking one token is rare. "Log this user out everywhere" is common: after a password change, an account compromise, or an admin action.

How it works:

1. Store a version counter per user.
2. Put the current version into each token as a claim.
3. To invalidate all of a user's tokens, increment the counter.
4. Verifiers reject any token whose version is older than the current one.

The values rarely change, so they cache well. Some systems push changes to verifiers over a message bus.

A variant stores a per-user "valid after" timestamp and rejects any token whose `iat` is earlier.

## Logout is mostly a client event

Logging out usually means two things: the server revokes the refresh token, and the client discards the access token.

The access token itself stays valid until it expires. If someone copied it, it works for the remaining minutes. If that window isn't tolerable, use Strategy 2 or 3.

## Stale claims

Suppose a token says `role: admin` and you demote the user. The token still says admin until it expires.

For sensitive actions such as payment approvals or admin operations, check the current permission at the moment of the action.

> 💡 Put identity in the token, because it rarely changes. Be careful about putting authority in the token, because it changes more often.

## Key rotation is a lifecycle event too

Rotate keys on a schedule, not only after an incident. A process that runs every few months will work in an emergency; one that has never run probably won't.

Know your emergency move: if a signing key leaks, remove it from the JWKS.

✅ **Check yourself**

- **Q15.** Why is instant revocation impossible with purely stateless JWTs?
- **Q16.** When would you choose token versions over a jti denylist?

---

# 🧩 Module 9 · What Belongs in the Payload

🎯 **You will be able to:** design a lean, safe payload.

Every claim you add has three properties:

- it travels on every request,
- anyone holding the token can read it,
- it is frozen until expiry.

Those three facts decide what belongs.

## Good candidates

- A stable, opaque user ID in `sub`. Not an email address, which changes and is personal data.
- `iss`, `aud`, `exp`, `iat`, `jti`.
- A tenant or organization ID.
- Coarse roles or OAuth scopes that rarely change.

## Poor candidates

- Passwords, secrets, or API keys. Never.
- Personal data the recipient doesn't need.
- Permission lists hundreds of entries long.
- Anything that changes often, such as balances or subscription status.

## Size

A session ID might be 32 characters. A JWT with a handful of claims is usually several hundred bytes to over a kilobyte, and it grows with every claim.

Size causes three problems:

- The token travels on every request, often across several internal hops.
- Cookies cap at about 4 KB.
- Many servers and proxies reject requests whose headers exceed roughly 8–16 KB, often with a confusing 400 or 431 error.

ES256's 64-byte signature is a quarter the size of a 2048-bit RSA signature.

> 💡 When permissions are large, put a role or scope in the token and resolve the details in the service, with caching.

✅ **Check yourself**

- **Q17.** Why use an opaque ID in `sub` instead of the user's email?
- **Q18.** Name two problems caused by large tokens.

---

# 🧩 Module 10 · JWTs Between Services

🎯 **You will be able to:** design token flows inside a microservices system.

## Verifying at the gateway

Having the API gateway verify every incoming token is consistent and saves work downstream.

> ⚠️ It becomes dangerous when the gateway is the only check. If internal services trust anything on the internal network, an attacker who gets inside can call any service with no token at all. One common way in is SSRF in a single service.

Each service should verify too. With a cached public key, it costs a fraction of a millisecond.

## Passing the user's identity downstream

Say orders calls inventory on the user's behalf. Forwarding the original token is simple, but its audience is orders, not inventory. If inventory accepts it, you've disabled the audience check.

The cleaner approach is **token exchange** (RFC 8693):

1. Orders presents the user's token to the auth service.
2. The auth service returns a new, short-lived token with inventory as the audience.
3. The new token keeps the same subject and carries only the scopes inventory needs.

Each hop gets a token valid for that hop alone.

## Service-to-service without a user

Background jobs use the OAuth **client credentials** grant. The service authenticates as itself and receives a short-lived token naming itself as the subject.

Workload identity in Kubernetes and the major cloud platforms can issue these automatically, so services don't need long-lived secrets on disk.

## ID tokens are not access tokens

OpenID Connect issues two JWTs at login:

|               | ID token                     | Access token                  |
| ------------- | ---------------------------- | ----------------------------- |
| Tells         | The client app who logged in | An API what the caller may do |
| Audience      | The client app               | The API                       |
| Send to APIs? | No                           | Yes                           |

✅ **Check yourself**

- **Q19.** Why shouldn't orders forward the user's token to inventory unchanged?
- **Q20.** What's the difference between an ID token and an access token?

---

# 🧩 Module 11 · When Sessions Are the Better Choice

🎯 **You will be able to:** decide between JWTs, sessions, or a hybrid, and defend the choice.

JWTs solve one specific problem: many independent verifiers that shouldn't depend on a central lookup. Without that problem, you pay the costs without collecting the benefit.

## Choose sessions when

- **You have one application** behind one domain. A shared Redis returns a session in about a millisecond or less.
- **You need instant revocation:** banking, admin panels, healthcare.
- **Your client is a browser you control.** HttpOnly session cookies are the most battle-tested pattern on the web.
- **Permissions change often** and must take effect immediately.

## Choose JWTs when

- **Many services verify identity**, and none of them should depend on one session store being up.
- **Tokens cross trust boundaries.** Partners can fetch your public key but can't query your database.
- **You're implementing OAuth 2.0 or OpenID Connect.**
- **Services call each other** and need short-lived, scoped credentials.
- **Tokens are single-purpose**, such as email verification links or signed download URLs.

## Comparison

|                             | Server sessions           | JWT access tokens               |
| --------------------------- | ------------------------- | ------------------------------- |
| Where state lives           | Server-side store         | Inside the token                |
| Per-request cost            | Lookup in a shared store  | Local signature check           |
| Revocation                  | Immediate                 | At expiry, unless you add state |
| Permission changes          | Immediate                 | At next refresh                 |
| What the client sees        | An opaque ID              | Readable claims                 |
| Cross-service / third party | Needs access to the store | Needs only the public key       |
| Size per request            | Tens of bytes             | Often 1 KB or more              |
| Main failure modes          | Store outage              | Key leaks, verification bugs    |

## The hybrid most large systems land on

1. The **browser holds an opaque, HttpOnly session cookie**, issued by a BFF or the gateway.
2. At the edge, the session is **exchanged for a short-lived JWT**.
3. **Internal services verify that JWT locally**, checking audience and expiry.

The browser gets the security of sessions: nothing readable, and instant revocation at the edge. The backend gets the scaling of JWTs: no central lookup per hop. This is sometimes called the **phantom token** pattern.

✅ **Check yourself**

- **Q21.** Give two situations where sessions beat JWTs.
- **Q22.** In the hybrid pattern, what does the browser hold, and what do internal services see?

---

# 🧩 Module 12 · JWTs in a System Design Interview

🎯 **You will be able to:** give a one-minute auth answer that shows judgment.

"Users authenticate with JWT" is the answer interviewers hear most, and it shows very little. A strong answer covers four decisions:

1. **Name the model.** "An auth service issues short-lived access tokens signed with ES256. Services verify them locally with keys from a JWKS endpoint, so there's no lookup on the hot path."
2. **Address revocation before you're asked.** "Access tokens last 10 minutes. Refresh tokens are opaque and rotated with reuse detection, so disabling an account takes effect within one token lifetime. Admin actions check permissions live."
3. **Say where the token lives.** "The web app uses a BFF with an HttpOnly cookie, so tokens never reach browser JavaScript. Mobile apps keep refresh tokens in the platform keystore."
4. **Mention verification hygiene.** "Verifiers pin the algorithm and check issuer, audience and expiry."

> 💡 For a single application at modest scale, "I'd use server-side sessions here, and here's why" is often the stronger answer. Interviewers look for judgment, not a fashionable acronym.

**Likely follow-ups:**

- How do you log a user out everywhere? (Module 8)
- What if the signing key leaks? (Module 8)
- Why not put all permissions in the token? (Modules 8 and 9)

✅ **Check yourself**

- **Q23.** What's weak about "we use JWT for auth" as an interview answer?
- **Q24.** What should you say about revocation without being asked?

---

# 📋 Cheat Sheet

## Recommended defaults

| Decision              | Default                                                                                 |
| --------------------- | --------------------------------------------------------------------------------------- |
| Signing               | ES256 or EdDSA (RS256 is fine); HS256 only for a single self-contained app              |
| HMAC secret, if used  | 256+ random bits, kept in a secrets manager                                             |
| Access token lifetime | 5–15 minutes                                                                            |
| Refresh token         | Opaque, rotated on every use, with reuse detection                                      |
| Clock skew leeway     | 30–60 seconds                                                                           |
| Web storage           | BFF, or access token in memory plus refresh token in an HttpOnly Secure SameSite cookie |
| Mobile storage        | Keychain / Keystore                                                                     |
| Never                 | Tokens in URLs, tokens in logs, secrets in the payload                                  |

## Verification in one line

Allowlisted alg → trusted key by kid → signature → exp/nbf → iss → aud → typ → then claims.

## Glossary

| Term           | Meaning                                                            |
| -------------- | ------------------------------------------------------------------ |
| Claim          | A statement inside the payload, such as `sub` or `exp`             |
| JWS            | Signed token (RFC 7515); readable by anyone                        |
| JWE            | Encrypted token (RFC 7516)                                         |
| JWK / JWKS     | One public key in JSON / a published set of them                   |
| kid            | Key ID in the header, used to pick the verification key            |
| Bearer token   | Whoever holds it is treated as the owner                           |
| Access token   | Short-lived JWT sent to APIs                                       |
| Refresh token  | Long-lived credential used only to get new access tokens           |
| ID token       | OIDC token telling the client app who logged in                    |
| BFF            | Backend for Frontend; holds tokens server-side for a web app       |
| Token exchange | Swapping one token for another with a narrower audience (RFC 8693) |
| DPoP           | Binding a token to a client-held key (RFC 9449)                    |
| XSS            | Injected script running in your page                               |
| CSRF           | Another site triggering requests that carry your cookies           |

---

# 🎯 Scenario Quiz

- **S1.** A startup runs one monolith web app on one domain. They want JWTs "because they're modern." What do you advise?
- **S2.** A pentester changes a token's alg from RS256 to HS256, re-signs it, and the API accepts it. What happened, and what's the fix?
- **S3.** Support must lock a compromised account immediately. Access tokens last 15 minutes. What are your options?
- **S4.** The payments service accepted a token issued for the notifications service. Which check is missing?
- **S5.** Tokens carry 300 permissions, and some requests fail at the load balancer with a 431 error. Why, and what's the fix?
- **S6.** The security team finds access tokens in the web server's access logs. How did they likely get there, and what's the fix?

---

# 🔑 Answer Key

**Module questions**

- **Q1.** A record of each logged-in user, which must be looked up on every request.
- **Q2.** Yes. The payload is base64url-encoded, not encrypted, so anyone can decode it.
- **Q3.** `kid`. During rotation, two or more keys are valid at once, and `kid` tells the verifier which one to use.
- **Q4.** The token was created by a holder of the signing key, and it hasn't been modified since.
- **Q5.** All 20 services hold the secret that can forge tokens. Compromising any one of them compromises authentication everywhere.
- **Q6.** Tokens signed with the old key are still in circulation until they expire. Without the old public key, they'd fail verification.
- **Q7.** The short-lived access token keeps per-request checks stateless. The refresh token forces a stateful check with the issuer, which can refuse to issue new access tokens.
- **Q8.** Reuse alone proves the token was copied. Revoking the whole family locks out the attacker; the real user just signs in again.
- **Q9.** It can trigger requests from the victim's open page with the cookie attached. It can't read the token or take it to use elsewhere.
- **Q10.** No token ever reaches browser JavaScript, so there is nothing for XSS to steal.
- **Q11.** The header is written by whoever made the token, possibly an attacker. Trusting it enables `alg: none` and algorithm confusion.
- **Q12.** The audience (`aud`) check. It prevents a token issued for one service from being replayed against another.
- **Q13.** The vulnerable server uses its RSA public key as the HMAC secret. The attacker knows that key, so they can compute a matching HMAC.
- **Q14.** It is used before verification to find a key. If it flows into file paths or SQL, an attacker controls that lookup.
- **Q15.** Validity comes from the token's contents, which the server can't change after issuing. Checking revocation means checking state.
- **Q16.** When you usually need to revoke all of a user's tokens (password change, compromise) rather than one specific token.
- **Q17.** Emails change and are personal data that anyone with the token can read. An opaque ID is stable and reveals nothing.
- **Q18.** Any two of: more bandwidth on every hop; going over the 4 KB cookie limit; going over server header limits; stale data frozen until expiry.
- **Q19.** Its audience is orders. Inventory would have to accept tokens not meant for it, which defeats the audience check. Use token exchange instead.
- **Q20.** An ID token tells the client app who logged in, and its audience is the client. An access token authorizes calls to an API, and its audience is that API.
- **Q21.** Any two of: a single app on one domain; a need for instant revocation; a browser client you control; permissions that change often.
- **Q22.** The browser holds an opaque HttpOnly session cookie. Internal services see short-lived JWTs created at the edge.
- **Q23.** It names a technology but not the decisions: lifetimes, revocation, storage, verification, or whether sessions would be better.
- **Q24.** How long a disabled user keeps access (one access-token lifetime), how refresh rotation enforces that, and where you check permissions live.

**Scenarios**

- **S1.** Use server-side sessions with an HttpOnly cookie. With one app, the lookup cost is negligible, and they get instant revocation. Revisit if they split into many services or expose a public API.
- **S2.** Algorithm confusion. The server let the header choose HS256 and used its RSA public key as the HMAC secret. Fix: allowlist only RS256 and use a library with typed keys.
- **S3.** Revoke the refresh-token family so no new access tokens are issued. For immediate effect, also bump the user's token version or add the active `jti` to a denylist.
- **S4.** The audience check. Payments must reject any token whose `aud` doesn't include payments.
- **S5.** The token is too large for header limits. Put a role or scope in the token and resolve detailed permissions in the service, with caching.
- **S6.** Most likely tokens passed in query strings (for example, `?token=` for downloads or WebSockets), or full Authorization headers being logged. Fix: send tokens only in headers or cookies, redact them in logs, use short-lived single-purpose tokens where a URL is unavoidable, and rotate any exposed signing material if needed.

---

# 📚 Further Reading

- [RFC 7519 · JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 7515 · JSON Web Signature](https://datatracker.ietf.org/doc/html/rfc7515)
- [RFC 7518 · JSON Web Algorithms](https://datatracker.ietf.org/doc/html/rfc7518)
- [RFC 8725 · JWT Best Current Practices](https://datatracker.ietf.org/doc/html/rfc8725)
- [RFC 9068 · JWT Profile for OAuth 2.0 Access Tokens](https://datatracker.ietf.org/doc/html/rfc9068)
- [RFC 9700 · OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/rfc9700)
- [RFC 8693 · OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693)
- [RFC 9449 · DPoP](https://datatracker.ietf.org/doc/html/rfc9449)

# 🔗 Source

[JWT Authentication Explained: How It Works, Where It Fails, and When to Use It](https://designgurus.substack.com/p/jwt-in-90-minutes-a-backend-engineers) by Arslan Ahmad, _System Design Nuggets_, Oct 28, 2026. Read the original for the author's own text and diagrams. Modules 1–3 here explain its free-preview topics in my own words; Modules 4–12 are an independent reconstruction.
