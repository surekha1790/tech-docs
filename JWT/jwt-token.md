# JWT Authentication — A Beginner's Guide
### (with real examples from our e-commerce project)

This guide explains JSON Web Tokens (JWT) from scratch — what they are, why we use them, exactly how our system works, and what a production-ready setup looks like. No prior knowledge assumed.

---

## 1. What is a JWT? (the simple version)

Imagine going to a concert. At the entrance you show your ticket once, and they give you a **tamper-proof wristband**. After that, you don't show your ticket again — you just flash the wristband at every door, and staff instantly know you're allowed in. Nobody has to call the box office to check.

A **JWT is that wristband.** It's a small string the server gives you after you log in, and you show it on every request. The server can instantly trust it *without looking anything up in a database*, because the wristband is **signed** and can't be faked.

### What it looks like

A JWT is one long string with **three parts separated by dots**:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI0MiIsImVtYWlsIjoicHJpeWFAeC5jb20iLCJyb2xlcyI6IkNVU1RPTUVSIn0.K3jd8sH2...signature
     └── HEADER ──┘ └──────────────── PAYLOAD ────────────────┘ └── SIGNATURE ──┘
```

- **Header** — says how it's signed (e.g., algorithm `HS256`).
- **Payload** — the actual data ("claims"): who you are, your roles, when it expires. In our system: `sub` (user id), `email`, `name`, `roles`, `userType`.
- **Signature** — a cryptographic seal created with a **secret key**. If anyone changes even one character of the payload, the signature no longer matches and the token is rejected.

**Important beginner point:** a JWT is **signed, not encrypted.** Anyone can read the payload (it's just Base64 — paste it into [jwt.io](https://jwt.io) to see). The signature doesn't *hide* the data; it *proves the data wasn't tampered with*. So never put secrets (passwords, card numbers) inside a JWT.

---

## 2. Why do we need it? (the problem it solves)

### The old way: server-side sessions

Traditionally, when you logged in, the **server remembered you** — it stored a "session" in its memory or a database and gave you a session id. On every request, the server looked up that session to know who you were.

This breaks down in a system like ours, which has **~14 microservices** (auth, order, payment, inventory, shipping, …). If every service had to look up a shared session store on every request, that store becomes a **bottleneck and a single point of failure**, and the services become tightly coupled to it.

### The JWT way: stateless authentication

With JWT, the server **doesn't remember anything.** The token itself carries your identity, and it's signed, so *any* service can verify it on its own using the shared secret — **no database lookup, no network call.** This is called **stateless** authentication.

This is exactly why JWT fits microservices:

- The gateway can verify a token in microseconds, locally.
- Services scale horizontally — no shared session state to synchronize.
- Adding a new service doesn't require wiring it into a session store.

The trade-off (we'll return to this): because nobody "remembers" the token, you can't easily *un-issue* one before it expires. That's why we also use **refresh tokens** stored in a database.

---

## 3. The two tokens: access vs refresh

Our system gives you **two** tokens at login, and understanding the difference is the key to the whole design.

| | **Access token** | **Refresh token** |
|---|---|---|
| What it is | A signed **JWT** | An opaque random string (not a JWT) |
| Purpose | Sent on **every** API request to prove who you are | Used **only** to get a new access token |
| Lifespan | **Short** (24h in dev, 1h in prod) | **Long** (7 days) |
| Stored in DB? | **No** (stateless) | **Yes** (`refresh_tokens` table) |
| Can be revoked? | Not directly (just expires) | **Yes** (we can mark it revoked) |

**Why two tokens?** It's a security-vs-convenience balance:

- If the access token is stolen, it's only useful for a short time (then it expires).
- But we don't want to make you log in with your password every hour — so the long-lived refresh token quietly gets you a fresh access token in the background.
- And because the refresh token lives in our database, we *can* cancel it instantly (on logout or password change) — something a pure stateless JWT can't do.

---

## 4. The JWT flow, step by step

Here's the whole lifecycle at a glance:

```
   ┌─────────┐   1. login (email + password)      ┌──────────────┐
   │         │ ─────────────────────────────────▶ │ auth-service │
   │ Client  │ ◀───────────────────────────────── │  (issuer)    │
   │(browser │   2. { accessToken, refreshToken }  └──────────────┘
   │ /mobile)│
   │         │   3. GET /api/orders                ┌──────────────┐
   │         │      Authorization: Bearer <access> │ api-gateway  │
   │         │ ─────────────────────────────────▶ │ (validator)  │
   │         │                                     └──────┬───────┘
   │         │                          4. validate token │ adds X-User-* headers
   │         │                                            ▼
   │         │                                     ┌──────────────┐
   │         │ ◀───────────────────────────────── │order-service │
   │         │   5. order data                     │ (trusts hdrs)│
   │         │                                     └──────────────┘
   │         │   6. access token expired (401)
   │         │   7. POST /auth/refresh (refreshToken)  → new access + new refresh
   └─────────┘
```

1. **Log in** with email + password → auth-service checks them.
2. auth-service returns an **access token** and a **refresh token**.
3. The client stores them and sends the **access token** on every request in the header `Authorization: Bearer <token>`.
4. The **gateway validates** the token (signature + not expired).
5. If valid, the request reaches the service, which returns data.
6. After a while the access token **expires** → the next request gets a **401 Unauthorized**.
7. The client silently calls **/auth/refresh** with the refresh token → gets a fresh access token (and a new refresh token) → retries. You never notice.

---

## 5. A real-time example flow (Priya buys milk)

Let's follow a real customer, **Priya**, end to end.

### Step 1 — Priya logs in

Her app sends:

```http
POST /api/auth/login
{ "email": "priya@example.com", "password": "hunter2" }
```

auth-service verifies the password, then builds a signed access token (this is the actual code):

```java
// auth-service: JwtService.generateAccessToken()
return Jwts.builder()
    .subject(user.getId().toString())      // sub = "42"
    .claim("email",    user.getEmail())    // priya@example.com
    .claim("name",     user.getFullName()) // Priya
    .claim("roles",    user.getRolesAsString())  // "CUSTOMER"
    .claim("userType", user.getUserType().name()) // CUSTOMER
    .issuedAt(now)
    .expiration(expiry)                    // now + 1 hour (prod)
    .signWith(signingKey)                  // HS256 with the shared secret
    .compact();
```

It also creates a refresh token — an opaque random string saved to the DB:

```java
// auth-service: JwtService.generateAndSaveRefreshToken()
refreshTokenRepository.revokeAllByUserId(user.getId()); // revoke old sessions
String tokenValue = UUID.randomUUID() + "-" + UUID.randomUUID();
// saved to refresh_tokens with a 7-day expiry
```

Priya's app receives:

```json
{
  "accessToken":  "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI0MiIsInJvbGVzIjoiQ1VTVE9NRVIifQ.K3jd...",
  "refreshToken": "6f1c2a90-...-b4e2-...-9d77"
}
```

### Step 2 — Priya places an order

Her app calls the order API, attaching the access token:

```http
POST /api/orders
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI0MiI...
{ "items": [ { "productId": 101, "quantity": 2 } ] }
```

### Step 3 — The gateway checks the wristband

The request first hits **api-gateway**, whose `JwtAuthenticationFilter` runs on every route. It:

1. Pulls the token out of the `Authorization: Bearer ...` header.
2. **Validates the signature and expiry locally** using the same shared secret — *no call to auth-service*:

   ```java
   // api-gateway: JwtAuthenticationFilter
   Claims claims = jwtUtil.validateAndExtractClaims(token);
   ```
3. Extracts Priya's identity from the claims and **injects it as plain headers**, then removes the raw token:

   ```java
   ServerHttpRequest enriched = request.mutate()
       .header("X-User-Id",    claims.getSubject())   // "42"
       .header("X-User-Roles", "CUSTOMER")
       .header("X-User-Type",  "CUSTOMER")
       .headers(h -> h.remove(HttpHeaders.AUTHORIZATION)) // strip the token
       .build();
   ```

If the token were missing, tampered, or expired, the gateway would stop right here and return **401** — the request never reaches order-service.

### Step 4 — order-service trusts the gateway

order-service never parses a JWT. It simply reads the header the gateway added:

```java
// order-service: OrderController
String userId = httpRequest.getHeader("X-User-Id");   // "42"
request.setCustomerId(userId);
```

The order is created for customer `42`, and Priya gets her confirmation. **The token was validated exactly once (at the gateway), and every downstream service just trusts the `X-User-*` headers.**

### Step 5 — An hour later, the token expires

Priya opens the app again. Her (now expired) access token is sent, and the gateway responds:

```json
{ "success": false, "status": 401, "message": "Token has expired. Please login again to get a new token." }
```

Her app catches the 401 and **silently refreshes** (this is exactly what our `customer-ui` axios interceptor does):

```http
POST /api/auth/refresh
{ "refreshToken": "6f1c2a90-...-9d77" }
```

auth-service validates the refresh token against the DB and **rotates** it:

```java
// auth-service: JwtService.rotateRefreshToken()
oldToken.setRevoked(true);                 // old refresh token is now dead
return generateAndSaveRefreshToken(...);   // brand-new refresh token issued
```

Priya's app gets a fresh access token + fresh refresh token, retries the original request, and she never even noticed she was logged out.

### Step 6 — Priya logs out

On logout, auth-service revokes her refresh token in the DB. Her old access token still *technically* works until it expires (it's stateless), which is why we keep it short-lived — and why a nightly `TokenCleanupScheduler` deletes expired refresh-token rows to keep the table lean.

---

## 6. How the pieces fit in our project

| Component | Role | Analogy |
|---|---|---|
| **auth-service** | The **issuer** — creates access tokens, stores/rotates refresh tokens, is the only one that checks passwords | The box office that prints wristbands |
| **api-gateway** | The **validator** — checks every access token, adds `X-User-*` headers, blocks bad tokens | The bouncer at the door |
| **order/payment/etc.** | The **trusters** — read `X-User-*` headers, never parse JWTs themselves | Staff who trust the wristband |
| **shared secret** | The signing key both auth-service and gateway hold, from config-server | The secret ink pattern on the wristband |

Key design choices in our code:

- **Signed with HS256** using one shared secret (from config-server), so the gateway can validate tokens without ever calling auth-service.
- **Access token is stateless** (never stored); **refresh token is stateful** (in `refresh_tokens`), so we get the best of both — fast validation *and* the ability to revoke.
- **Refresh-token rotation**: every refresh revokes the old token and issues a new one, and we revoke all of a user's tokens on login/logout/password change (single-session policy).
- **Validate once, at the edge**: only the gateway parses the JWT; services trust its headers. This keeps every service simple.

---

## 7. Production-ready checklist

Our design already does a lot right. Here's the full picture — what's solid, and what to harden before heavy production use.

### Already good ✔
- Short-lived access token + long-lived refresh token.
- Refresh-token **rotation** and **DB revocation** (logout/password-change kills sessions).
- **Edge validation** at the gateway; downstream services stay simple.
- Secret injected from **config/env var**, not hard-coded.
- Nightly cleanup of expired tokens.

### Harden for production 🔧

1. **Shorten the access token** — 24h (our dev value) is too long. Use **15–60 minutes** in prod (the config already switches to 1h). Shorter = less damage if one leaks.

2. **Switch HS256 → RS256 (asymmetric).** Today auth-service *and* the gateway share the same secret (symmetric). If any service leaks it, anyone can forge tokens. With **RS256**, auth-service holds a **private key** and signs; everyone else verifies with a **public key** they can't forge with. Publish the public key via a **JWKS endpoint** so keys can be rotated cleanly.

3. **Store secrets in a vault** — HashiCorp Vault or AWS Secrets Manager, not plain text in config. Rotate them periodically.

4. **Lock down the network.** Because downstream services *trust* the `X-User-Id` header, they must be **unreachable except through the gateway** (private network / service mesh / mTLS). Otherwise an attacker who reaches a service directly could just set `X-User-Id: 42` and impersonate anyone. This is the single most important production control for this architecture.

5. **HTTPS everywhere** — tokens in headers must never travel over plain HTTP.

6. **Client storage.** Storing tokens in browser `localStorage` exposes them to XSS. Prefer **httpOnly + Secure cookies** for the refresh token so JavaScript can't read it.

7. **Refresh-token reuse detection.** Since we rotate refresh tokens, if an *already-used* (revoked) refresh token is presented again, that's a red flag for theft — revoke the entire token family and force re-login.

8. **Validate `issuer` and `audience`** claims (our tokens set them) and allow only a **small clock-skew** so tokens from other systems or slightly-off clocks are rejected cleanly.

9. **Instant revocation option.** Pure JWT can't be revoked before expiry. If you need immediate kill-switch for access tokens (e.g., ban a user right now), either keep the access token very short or add a small denylist checked at the gateway.

---

## 8. Quick glossary

- **JWT** — a signed token carrying identity claims; verifiable without a database.
- **Claim** — a field inside the token (`sub`, `email`, `roles`, `exp`…).
- **Access token** — short-lived JWT sent on every request.
- **Refresh token** — long-lived credential used only to get new access tokens; stored in our DB.
- **HS256 / RS256** — signing algorithms; HS256 = one shared secret, RS256 = private/public key pair.
- **Bearer token** — sent as `Authorization: Bearer <token>`; "whoever bears it is trusted."
- **Stateless** — the server keeps no session; the token carries everything.
- **Rotation** — replacing the refresh token on every use so a stolen one becomes useless.
- **Claims validation** — checking signature, expiry, issuer, and audience before trusting a token.

---

### The one-paragraph summary

When Priya logs in, **auth-service** hands her a short-lived signed **access token** (a stateless JWT) and a long-lived **refresh token** (stored in our database). She sends the access token on every request; the **api-gateway** validates it once and forwards her identity to services as `X-User-*` headers, so no other service needs to understand JWTs. When the access token expires, her app silently trades the refresh token for a new one (rotating it for safety). This gives us **fast, stateless, scalable authentication across all our microservices**, while still letting us **revoke sessions** when needed — and going to production mainly means shorter access tokens, asymmetric keys, a secrets vault, and a locked-down network so the `X-User-*` trust can't be abused.
