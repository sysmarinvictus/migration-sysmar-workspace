---
name: scaffold-target
description: One-time creation of the receituario-modern monorepo skeleton — Spring Boot 3 backend (Java 21, JPA, Spring Security+JWT, Flyway, OpenAPI, common infrastructure), React+TS+Vite frontend, Testcontainers/RestAssured test harness, docker-compose, and CI. Run once before migrating any slice.
---

# scaffold-target

Creates `/mnt/s/projetos/receituario-modern/` (sibling of the GeneXus tree — never inside it). Idempotent:
if it already exists, report what's present and only fill gaps. Read ARCHITECTURE.md §3, §7, §10.

## What to create
**Root:** `pom.xml` (Java 21, Spring Boot 3.4.x parent), `docker-compose.yml` (PostgreSQL 16 +
Adminer for dev; never points at production), `.gitignore`, `README.md`, `.github/workflows/ci.yml`.

**backend/** (`br.gov.mandaguari.saude`):
- `Application.java`; profiles `local`/`test`/`parity` in `application*.yml` (DB creds via env vars,
  **never** committed literals — the secret-scan hook enforces this).
- `config/`: `SecurityConfig` (stateless JWT, CORS to the frontend origin, public auth endpoints,
  everything else authenticated), `JwtService`, `OpenApiConfig`, `JpaAuditingConfig`.
- `common/error/`: `GlobalExceptionHandler` → RFC-7807 `ProblemDetail`; domain exception base.
- `common/audit/`: `@Auditable` + an audit writer that records PHI access (replaces `SAU_LOG`).
- `common/validation/`: `@CPF`, `@CNPJ`, `@CNS` constraints (port `psau_val_cnpjcpf`, `PSAU_VER_CNS`).
- `common/persistence/`: `BaseEntity` and a Hibernate physical naming strategy that **preserves
  GeneXus uppercase table/column names** (no camelCase translation at the DB layer).
- `security/`: `AuthController` (login → JWT, refresh), `UserDetailsService` over `SAU_USU`/`SYS_PES`
  (password scheme TBD — see the auth slice open question).
- `db/migration/V1__baseline.sql`: the introspected current schema; configure Flyway
  `baseline-on-migrate: true` so the existing populated DB is adopted, not recreated. If the DB
  isn't reachable to introspect, leave a documented placeholder and a TODO to generate it.
- `src/test/.../AbstractIntegrationTest`: shared `@SpringBootTest` + `@Container PostgreSQLContainer`
  base with Flyway + RestAssured wired; a `ParityTestBase` tagged `parity`.

**frontend/** (Vite + React 18 + TS): `package.json`, `tsconfig` (strict), Tailwind + shadcn/ui
setup, `orval.config.ts` (generate the client from backend OpenAPI), `src/lib/` (api client, query
client, auth context, pt-BR masks for CPF/CNS/CEP), `src/app/` (router, layout, login page), an empty
`src/features/`.

## After scaffolding
1. Verify structure (`tree`/`find`). If JDK 21 + Maven exist, run `mvn -q -pl backend compile`; if not,
   print the SDKMAN install commands (ARCHITECTURE §10). If npm is present, `npm --prefix frontend i`.
2. `git init` the new repo (the legacy tree is separate and stays untracked here).
3. Report what was created and the exact next commands (`/gx-extract <leaf-transaction>`).

## Guardrails
- Everything goes under `receituario-modern/`. The regen-guard hook blocks the GeneXus/Tomcat trees.
- No real credentials or PHI in any committed file. DB URL/user/password come from env vars.
