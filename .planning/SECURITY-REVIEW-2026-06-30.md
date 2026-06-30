# Security Review — Wave-0 Auth / RBAC / Bridge (2026-06-30)

Adversarial SAST-style review of the security-critical code shipped in the recent sessions (SAU_USU
auth, password bridge, RBAC resolver, perfil/programa CRUD, JWT/Spring Security config). Aikido MCP was
not yet configured (needs `/aikido:setup <key>` + restart), so this is a manual review; re-run the Aikido
scan once configured for the dependency/CVE/SCA layer.

**Verdict at review time:** 1 CRITICAL, 1 HIGH, 4 MEDIUM, 2 LOW. Injection, JWT sig/type handling,
secret serialization, the password bridge, and audit PHI handling were **clean**.

## Fixed this session

| ID | Sev | Finding | Fix |
|----|-----|---------|-----|
| **C1** | CRITICAL | Default Spring profile was `local` (`application.yml`), so a deploy that forgets `SPRING_PROFILES_ACTIVE` boots the **`admin/admin123` dev backdoor** (`DevUserDetailsService`) + the committed dev JWT secret → full admin via known creds / forgeable tokens. | Default profile now **empty** (`${SPRING_PROFILES_ACTIVE:}`) → no profile = SECURE path (real `SauUsuUserDetailsService`, `JWT_SECRET` mandatory, no fallback → fails fast). Dev/test set the profile explicitly (dev-up.sh `--spring.profiles.active=local`; tests `@ActiveProfiles("test")`). |
| **H2** | HIGH | Same root cause: committed dev JWT secret (`application-local.yml`) reachable in prod via the `local` default → token forgery (stateless filter trusts `roles` claim verbatim). | Closed by C1. Backstopped by **`ProductionSecurityGuard`** (`@Profile("!local & !test")`): aborts startup if the JWT key looks like a placeholder (`insecure`/`change-me`/`dev-local`), is < 32 bytes, or if the `DevUserDetailsService` bean is present. Unit-tested (4). |
| **M5** | MEDIUM | `/api/usuarios/lookup` had a method-level `@PreAuthorize("isAuthenticated()")` overriding the class-level `SAUDE_ADMIN`, exposing `UsuNom` (PII) + enabling user enumeration to any authenticated user (LGPD). | Removed the override → inherits `SAUDE_ADMIN`. The user picker is admin-only. |

## Accepted / backlog (documented, not fixed now)

| ID | Sev | Finding | Disposition |
|----|-----|---------|-------------|
| M3 | MEDIUM | Roles are read from the JWT and not revalidated until expiry (≤30m); a demoted/blocked admin keeps access for the access-token TTL. Refresh path DOES re-check (`requireUsableAccount`). | Accept the 30m window for now; consider a `jti`/token-version denylist for high-value revocations. Backlog. |
| M4 | MEDIUM | `PermissionResolver` profile-precedence can over-DENY (fail-secure) if a PRFCON row is missing — not a bypass. Correctness depends on the mined `pisauthorized` rule. | Gate on the RBAC parity run (SAU_RBAC OQ4) before wiring. Not a vulnerability. |
| M6 | MEDIUM | Fine-grained `@rbac.can` engine is built but **not wired** — enforcement still rests on coarse `hasRole`. | Tracked: SAU_RBAC OQ1 (endpoint-by-endpoint wiring, gated on OQ4 parity). |
| L7 | LOW | `nextUsuCod()` / `findMaxId()+1` ID allocation is race-prone (concurrent creates → PK collision); `findAll(unpaged)` full-load in UsuarioService. | Backlog: DB sequence / `SELECT max` aggregate. (Also SAU_USU OQ14.) |
| L8 | LOW | No rate-limiting on `/auth/login` / `/auth/change-password`; no auto-lockout (UsuBloq is admin-set). bcrypt + generic-failure already prevent enumeration. | Backlog: add throttling (bucket4j / Spring Security max-attempts). |

## Verified clean (no action)

- **SQL/JPQL injection:** every `@Query`/`nativeQuery` uses bound `:params`; no string concatenation of user input. LIKE-wildcard only (harmless).
- **JWT:** `parseSignedClaims` rejects `alg:none`/unsigned; refresh-vs-access `typ` confusion prevented both ways; expiry enforced.
- **Secret leakage:** `UsuSen`/`UsuKey` `@JsonIgnore`, absent from DTOs/mappers/`toString()`; no logger emits secrets/PII; `{noop}` sentinel can't bcrypt-match (no DelegatingPasswordEncoder).
- **Password bridge:** dry-run default, never logs plaintext/hashes, self-verifies, parameterized UPDATE, idempotent, env creds.
- **Defensive catches** (PerfilService/UsuarioService/PermissionResolver) all fail toward **deny/not-referenced**, never toward allow.
- **Audit PHI:** `AuditService` nulls all name/patient columns; sensitive writes audited; `AuditoriaController` admin read-only. `@EnableMethodSecurity` on; state-changing endpoints all carry `@PreAuthorize`; CSRF-disable correct for stateless bearer API.

## Follow-up
- Configure Aikido (`/aikido:setup <key>` + restart) and run `aikido:scan` for the dependency/CVE/SCA layer this manual review does not cover.
- ✅ **`prod` profile added** (`application-prod.yml`): API docs/Swagger disabled, actuator reduced to `/health` (no details), SQL logging off, Flyway validates the populated DB, CORS + all secrets required from env (no fallbacks). Smoke-tested against the snapshot: dev secret → `ProductionSecurityGuard` aborts; strong secret → boots, `/health` UP, `/swagger-ui.html` 404, no dev backdoor bean. **Still TODO:** a deploy manifest (Dockerfile/compose/CI) that pins `SPRING_PROFILES_ACTIVE=prod` and supplies the env.
