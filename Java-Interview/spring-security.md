### * How user login flow and spring security works ?
- When user tries to login, first verify whether user is active or not.
- If active then authenticate the user by
  ```
  authenticationManager.authenticate(
    new UsernamePasswordAuthenticationToken(
      user.getId().toString(),  // principal is UUID
      request.getPassword()
    )
  );
  ```
- ```authenticate()``` method hands off the validation to ```DaoAuthenticationProvider```
  ```
    DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
    provider.setUserDetailsService(userDetailsService);
    provider.setPasswordEncoder(passwordEncoder());
    return provider;
  ```
- It fetches the password using userDetailsService and check the given password against encoded password from DB

### * What is SecurityContextHolder ?
- It holds the current authenticated user.
- ```AuthenticationFilter``` builds ```Authentication``` object and set it to ```SecurityContextHolder.getContext().setAuthentication(authToken);```
- Subsequent requests like ```@PreAuthorize``` gets logged in user details from holder.
- Stored in threadlocal per request and should be cleared ```SecurityContextHolder.clearContext();```

## 1. Core architecture & the filter chain

### Q1. How does Spring Security work internally?
Spring Security is fundamentally a **chain of servlet Filters**, not interceptors or AOP. The flow:

- A single **`DelegatingFilterProxy`** is registered in the servlet container. It delegates to a Spring-managed bean named `springSecurityFilterChain`.
- That bean is a **`FilterChainProxy`**, which holds one or more **`SecurityFilterChain`s**. Each `SecurityFilterChain` is a request matcher + an **ordered list of security filters**.
- For each request, `FilterChainProxy` picks the first matching `SecurityFilterChain` and runs its filters in order.

Each filter has one job: `SecurityContextHolderFilter` (restores the context), `CsrfFilter`, `UsernamePasswordAuthenticationFilter` (form login), `BearerTokenAuthenticationFilter` (OAuth2), `ExceptionTranslationFilter` (turns auth exceptions into 401/403), and finally the **`AuthorizationFilter`** (enforces access rules). Understanding that **"it's just an ordered filter chain"** is the single most important mental model.

### Q2. Why `DelegatingFilterProxy`?
The servlet container instantiates filters *before* the Spring context exists, and container-created filters aren't Spring beans (no DI, no lifecycle). `DelegatingFilterProxy` is a thin container-level filter that simply forwards to a **Spring-managed** filter bean — bridging the servlet world and the Spring context so the real security filters can be full beans with dependency injection.

### Q3. Authentication vs Authorization — precisely.
**Authentication** = *who are you?* (verify identity, produce an `Authentication` with authorities). **Authorization** = *are you allowed to do this?* (check those authorities against a rule). Authentication happens early in the chain; authorization happens last (`AuthorizationFilter`). A 401 is "not authenticated"; a 403 is "authenticated but not permitted."

---

## 2. Authentication internals

### Q4. Walk through the authentication flow end to end.
1. An authentication filter extracts credentials (form fields, a Bearer token, etc.) and builds an unauthenticated `Authentication` token.
2. It calls **`AuthenticationManager.authenticate(token)`**. The default `AuthenticationManager` is a **`ProviderManager`** holding a list of **`AuthenticationProvider`s**.
3. `ProviderManager` asks each provider "can you handle this token type?" The **`DaoAuthenticationProvider`** handles username/password: it calls `UserDetailsService.loadUserByUsername()` to fetch the user + stored hash, then `PasswordEncoder.matches(raw, hash)` to verify.
4. On success it returns a **fully authenticated** `Authentication` (principal + authorities, credentials erased). On failure it throws `AuthenticationException` (e.g., `BadCredentialsException`).
5. The filter stores the result in the **`SecurityContextHolder`**.

*In this project:* `AuthService.login()` calls `authenticationManager.authenticate(...)`; `ApplicationConfig` wires a `DaoAuthenticationProvider` with `UserDetailsServiceImpl` + `BCryptPasswordEncoder(12)`. The password check is `BCryptPasswordEncoder.matches()`, never a manual comparison.

### Q5. `SecurityContextHolder` — how does it work, and what's the concurrency risk?
It holds the current `SecurityContext` (→ `Authentication` → principal/credentials/authorities) in a **`ThreadLocal`** by default (`MODE_THREADLOCAL`). Because one request = one thread, the authenticated user is visible anywhere in that request without passing it around. **The risk:** thread pools **reuse threads**, so the context *must be cleared* at request end (Spring's `SecurityContextHolderFilter` does this in a `finally`) — otherwise one request's identity leaks into the next. For async work, use `MODE_INHERITABLETHREADLOCAL` or `DelegatingSecurityContextExecutor` to propagate the context to child threads.

### Q6. `UserDetailsService` vs `AuthenticationProvider` — when to implement which?
Implement **`UserDetailsService`** when you just need to *load a user* and let Spring's `DaoAuthenticationProvider` do the password check the standard way. Implement a custom **`AuthenticationProvider`** when the *authentication logic itself* is non-standard — OTP, LDAP, an external IdP, multi-factor, or a token you validate yourself. The provider gives you full control over how the `Authentication` is verified and built.

### Q7. How is a password verified without storing plaintext?
Passwords are stored as **adaptive one-way hashes** (BCrypt here, cost 12). The hash embeds its own **salt + cost**, so `matches(raw, hash)` re-hashes the input with that salt and compares — you never decrypt. A senior should mention **`DelegatingPasswordEncoder`** (the Boot default, prefix like `{bcrypt}`) which allows **multiple encoders and seamless upgrades**, and **`upgradeEncoding`** to re-hash on successful login when the algorithm/cost changes.

---

## 3. Authorization

### Q8. Difference between a **role** and an **authority**, and the `ROLE_` prefix trap.
Internally there are only **authorities** (`GrantedAuthority` strings). A "role" is just an authority conventionally prefixed with **`ROLE_`**. `hasRole("ADMIN")` checks the authority `ROLE_ADMIN` (Spring adds the prefix); `hasAuthority("ADMIN")` checks `ADMIN` literally. The classic bug is storing roles as `ADMIN` but checking `hasRole` (→ `ROLE_ADMIN`) and getting silent 403s. *In this project:* `/auth/roles/**` uses `hasRole("SUPER_ADMIN")` → requires authority `ROLE_SUPER_ADMIN`.

### Q9. URL-based vs method-level authorization — when to use each?
**URL-based** (`authorizeHttpRequests().requestMatchers(...).hasRole(...)`) is coarse, centralized, and great for broad rules ("all `/admin/**` needs ADMIN"). **Method security** (`@PreAuthorize`, `@PostAuthorize`, `@Secured`, `@RolesAllowed`) is fine-grained, sits next to business logic, and supports **SpEL** — e.g. `@PreAuthorize("hasRole('ADMIN') or #order.ownerId == authentication.name")` for ownership checks. Use URL rules for the perimeter and method security for domain-specific rules; they compose as defense in depth. Enable with `@EnableMethodSecurity` (SS6).

### Q10. `@PreAuthorize` vs `@PostAuthorize` vs `@Secured` vs `@RolesAllowed`?
- **`@PreAuthorize`** — SpEL, evaluated *before* the method (most powerful/common).
- **`@PostAuthorize`** — SpEL, *after* the method, can inspect the return value (`returnObject`) — e.g., only return an object the caller owns.
- **`@Secured`** — legacy, simple role list, no SpEL.
- **`@RolesAllowed`** — the JSR-250 standard equivalent of `@Secured`.
Method security works via **Spring AOP proxies**, so the usual proxy caveats apply (self-invocation bypasses it; the bean must be called through its proxy).

### Q11. How does authorization actually get enforced?
The last filter, **`AuthorizationFilter`** (SS6; formerly `FilterSecurityInterceptor`), consults an **`AuthorizationManager`** (SS6; replaced the older `AccessDecisionManager`/voters). The manager evaluates the configured rules against the current `Authentication`'s authorities and returns grant/deny. Deny → `AccessDeniedException` → `ExceptionTranslationFilter` → **403** (or 401 if not authenticated).

### Q12. 401 vs 403 — who produces them?
`ExceptionTranslationFilter` catches security exceptions. An **`AuthenticationException`** (no/invalid credentials) → it invokes the **`AuthenticationEntryPoint`** → **401**. An **`AccessDeniedException`** (authenticated, insufficient rights) → the **`AccessDeniedHandler`** → **403**. You customize these for API-friendly JSON error bodies.

---

## 4. Stateless / JWT / token auth

### Q13. How do you implement JWT authentication in Spring Security?
Two idiomatic ways:
1. **Custom filter** (what many hand-rolled apps do): a `OncePerRequestFilter` that reads the `Authorization: Bearer` header, validates the JWT, builds an `Authentication`, and sets the `SecurityContextHolder` — placed before `UsernamePasswordAuthenticationFilter`. Set `SessionCreationPolicy.STATELESS`. *This project's auth-service uses this pattern.*
2. **Spring Security OAuth2 Resource Server** (the recommended way): `http.oauth2ResourceServer(o -> o.jwt(...))` with a `JwtDecoder`. Spring handles extraction, signature/claim validation via a `JwtDecoder` (JWKS for RS256), and authority mapping — far less custom code and fewer footguns. A senior should recommend this over a hand-rolled filter unless there's a reason not to.

### Q14. Why `SessionCreationPolicy.STATELESS` for APIs?
So Spring Security **never creates or uses an `HttpSession`** — no `JSESSIONID`, no server-side session state. Each request is authenticated **solely from the token**, which is what makes the service horizontally scalable (any instance can serve any request) and is the whole point of JWT. It also means the `SecurityContext` isn't persisted between requests — it's rebuilt from the token every time.

### Q15. What must you validate on a JWT, and what are the classic pitfalls?
Validate the **signature** (with the right key/algorithm), **expiry (`exp`)**, and ideally **issuer (`iss`)** and **audience (`aud`)**. Pitfalls a senior must name: the **`alg=none`** attack (never let the token dictate the algorithm — pin it), **algorithm confusion** (RS256 verified as HS256 using the public key as an HMAC secret), a **leaked HS256 secret** (symmetric — anyone with it can forge; prefer **RS256** so only the issuer signs), no expiry, and the fact that **stateless JWTs can't be revoked** before expiry (mitigate with short TTL + refresh-token revocation or a denylist).

### Q16. Access vs refresh tokens — the design reasoning.
Access token = short-lived, stateless, sent every request (limited blast radius, no DB check). Refresh token = long-lived, **stored server-side so it's revocable**, used only to mint new access tokens. You get a long session *and* revocation control without a DB lookup per request. Harden with **rotation** and **reuse detection**.

### Q17. In a microservices system, where do you validate the token?
Validate **once at the API gateway** (edge) and forward identity to downstream services (e.g., `X-User-*` headers), so services don't each parse JWTs. The critical caveat: those services now **trust a header**, so they must be **network-isolated** (private network / mTLS) so nobody can reach them directly and spoof `X-User-Id`. *This project does exactly this.* Alternatives: propagate the token itself and make each service an OAuth2 resource server, or use a service mesh with mTLS identity.

---

## 5. Sessions, CSRF, CORS

### Q18. What is CSRF and when do you need protection?
CSRF tricks a logged-in user's browser into sending an unwanted request using **ambient credentials** (cookies) it sends automatically. Protection (a synchronizer/CSRF token the attacker can't read) is needed for **stateful, cookie/session-based** apps. For a **stateless JWT API** where the token is sent explicitly in a header (not auto-attached by the browser), CSRF doesn't apply, so it's commonly disabled — **but** if you move the token into an httpOnly **cookie**, CSRF is back and you need `SameSite`/CSRF tokens again. Know *why* you're disabling it, not just that you did.

### Q19. Session fixation and `SessionCreationPolicy` options.
`ALWAYS`, `IF_REQUIRED` (default), `NEVER` (don't create but use if present), `STATELESS` (never touch sessions). For stateful apps, Spring's **session fixation protection** (`changeSessionId`, default) issues a new session id on login to prevent an attacker fixing a known id. Also configurable: concurrent-session control (max sessions per user).

### Q20. How does CORS interact with Spring Security?
CORS must be handled **before** authorization (a blocked preflight would otherwise 401). Configure a `CorsConfigurationSource` and enable `http.cors(...)`; Spring's `CorsFilter` runs early and lets `OPTIONS` preflights through. Key gotcha: with **`allowCredentials=true`** (needed for cookies/Authorization), you **cannot** use `*` origins — you must list explicit origins. *This project's gateway sets `allowCredentials(true)` with explicit origins.*

---

## 6. Reactive & modern API

### Q21. How is security different in WebFlux (reactive)?
There's no servlet filter chain or `ThreadLocal` context. Instead: **`SecurityWebFilterChain`** built from `ServerHttpSecurity`, **`WebFilter`s** instead of servlet filters, and **`ReactiveSecurityContextHolder`** which stores the context in the **Reactor `Context`** (bound to the reactive pipeline, not a thread — because reactive work hops threads). *This project's API gateway is WebFlux, so its JWT filter is a reactive `GlobalFilter`/`WebFilter`, not a servlet filter.*

### Q22. What changed in Spring Security 6 / Boot 3 that a senior must know?
- **`WebSecurityConfigurerAdapter` is removed** — you now expose a **`SecurityFilterChain` bean** and use the **lambda DSL** (`http.authorizeHttpRequests(auth -> ...)`).
- `authorizeRequests()` → **`authorizeHttpRequests()`**; `antMatchers`/`mvcMatchers` → **`requestMatchers`**.
- **`AuthorizationManager`** replaced `AccessDecisionManager`/voters.
- `@EnableGlobalMethodSecurity` → **`@EnableMethodSecurity`** (which uses `AuthorizationManager` and enables `@PreAuthorize` by default).
- Requires **Jakarta EE** (`jakarta.*`) namespaces. Knowing the migration signals you've kept current.

### Q23. Multiple `SecurityFilterChain`s — why and how?
You can register several `SecurityFilterChain` beans with different `securityMatcher`s and `@Order` — e.g., a **stateless, no-CSRF chain** for `/api/**` and a **session-based, CSRF-on chain** for an admin/actuator UI, or different auth mechanisms per path. `FilterChainProxy` picks the first matching chain. This is the clean way to run distinct security policies in one app.

### Q23.1 What does it mean by below code snippet ?
```
 .authenticationProvider(authenticationProvider)
            .addFilterBefore(jwtAuthenticationFilter,
                    UsernamePasswordAuthenticationFilter.class);

```
- authenticationProvider -> is the bean which is configured to validate username and password
```@Bean
AuthenticationProvider authenticationProvider() {
    DaoAuthenticationProvider provider =
        new DaoAuthenticationProvider();
    provider.setUserDetailsService(userDetailsService);
    provider.setPasswordEncoder(passwordEncoder());
    return provider;
}
```
- JwtAuthFilter should run before UsernamePasswordAuthenticationFilter as jwtFilter extract details and set securityContextHolder in auth service.
---

## 7. Testing, hardening, war stories

### Q24. How do you test secured code?
Use **`spring-security-test`**: `@WithMockUser(roles="ADMIN")` / `@WithUserDetails` to set a `SecurityContext` for a test, and `SecurityMockMvcRequestPostProcessors` (`.with(jwt())`, `.with(user(...))`, `csrf()`) for MockMvc. Test both the **happy path and the deny path** (assert 401/403) — a senior tests that unauthorized access is actually blocked, not just that authorized access works.

### Q25. Common Spring Security vulnerabilities you've guarded against?
- **User enumeration** — return a generic "Invalid credentials" for both unknown-user and bad-password (this project does).
- **Privilege escalation via roles** — lock role-management to the highest role only (`/auth/roles/**` → SUPER_ADMIN here).
- **JWT footguns** — pin the algorithm, enforce `exp`/`iss`/`aud`, prefer RS256, protect the secret.
- **Header-trust spoofing** — network-isolate services that trust gateway `X-User-*` headers.
- **Over-permissive CORS / `permitAll`** — audit public paths; don't `permitAll` by accident.
- **Missing method-level checks** — URL rules alone miss internal calls; add `@PreAuthorize` for sensitive operations.

### Q26. How would you design auth for a fresh microservices platform today?
Central **IdP / OAuth2 Authorization Server** issuing **RS256** JWTs; services are **OAuth2 Resource Servers** validating via **JWKS** (no shared secret); short access tokens + rotating refresh tokens; **gateway** for edge validation and coarse authZ; **method security** for fine-grained rules; **mTLS/service mesh** for service-to-service identity; secrets in a **vault**; and audience-scoped tokens so a token for one API can't be replayed on another (this project added exactly that with admin vs customer audiences).

---

### How this maps to your project (quick reference)
- **Filter-based JWT auth** in auth-service (`OncePerRequestFilter` → `SecurityContextHolder`).
- **`DaoAuthenticationProvider` + `BCryptPasswordEncoder(12)`** for password checks.
- **Role rules**: `/auth/users/**` → ADMIN/SUPER_ADMIN, `/auth/roles/**` → SUPER_ADMIN.
- **Reactive `SecurityWebFilterChain`** + `GlobalFilter` at the gateway; validate once, forward `X-User-*`.
- **Stateless** access tokens (HS256 today — RS256 is the production upgrade), **DB-stored revocable** refresh tokens with rotation.
- **Audience scoping** enforced at the gateway for `/api/admin/**`.

