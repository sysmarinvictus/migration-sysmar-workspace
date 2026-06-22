# Receituário Migration — Architecture & Playbook

**Status:** active · **Owner:** migration factory · **Created:** 2026-06-01
**Strategy:** Strangler-fig, slice-by-slice · **DB:** keep existing PostgreSQL schema (introspect)

> This document is the single source of truth for the GeneXus → Spring Boot/React migration.
> Every factory agent and skill references the contracts and conventions defined here.
> It is hand-written planning — **not** GeneXus output — and is safe to edit.

---

## 1. Goal

Replace the GeneXus-generated webapp `Receituario` (Tomcat 8.5, ~34 transactions, LGPD-regulated
Brazilian municipal health prescription system) with a modern stack, **without a big-bang cutover**:

| Layer | From (GeneXus) | To (target) |
|-------|----------------|-------------|
| Language/runtime | Generated Java 1.5-era, flat default package, Tomcat 8.5 | **Java 21**, Spring Boot 3.4.x |
| Persistence | GeneXus cursors (`__default`), JDBC | **Spring Data JPA** over the *same* PostgreSQL schema |
| API | Servlet/HTML roundtrips, no REST | **REST** (OpenAPI 3), RFC-7807 errors |
| UI | Generated JSP-like HTML + jQuery 1.2.6 + applet print | **React 18 + TypeScript + Vite** |
| Auth | Custom session login, GeneXus URL encryption | **Spring Security + JWT** (stateless) |
| Tests | None | **JUnit 5 + Mockito** (unit), **Testcontainers + RestAssured** (integration), golden-master parity |
| Build | `GXJMake.exe` | **Maven** (backend), **npm/Vite** (frontend) |

**Non-negotiable constraints (from `.planning/codebase/CONCERNS.md`):**
- LGPD Art. 11 sensitive health data (PHI): patient names, CPF, CNS, diagnoses, prescriptions.
  No PHI in logs; audit every PHI read/write (replaces `SAU_LOG`).
- Portaria SVS/MS 344/98 controlled-substance prescription rules (`SAU_RECESP`) must be preserved
  exactly — required fields, two-via printout, retention. Verify with regulatory lead per slice.
- Secrets never committed (the GeneXus `client.cfg` leaked DB creds — see CONCERNS C1).
- **The GeneXus tree is read-only.** `Receituario/JavaModel/web/**` is regenerated; never edit it.
  A PreToolUse hook enforces this.

---

## 2. Strangler-fig model

```
                    ┌──────────────────────────────┐
   Browser ───────► │  Reverse proxy (per-route)   │
                    └───────────┬───────────┬──────┘
                  migrated routes│           │legacy routes
                                 ▼           ▼
                  ┌──────────────────┐  ┌─────────────────────────┐
                  │ receituario-     │  │ GeneXus webapp (Tomcat) │
                  │ modern (SB+React)│  │ /ReceituarioJavaEnv...  │
                  └────────┬─────────┘  └───────────┬─────────────┘
                           └──────────┬─────────────┘
                                      ▼
                       PostgreSQL  saude-mandaguari   ◄── single shared DB
                                   (one schema, both apps)
```

- **Both apps run against the same PostgreSQL DB.** Because we keep the schema, a migrated slice
  and the un-migrated GeneXus screens stay data-consistent during the transition.
- A slice is "cut over" by routing its URLs to the new app at the proxy. Until then the new
  endpoints exist but legacy serves users — enabling shadow/parity testing in production.
- Order of cutover follows the **dependency-ordered backlog** (`BACKLOG.md`): leaf reference
  tables first (especialidade, bairro, CEP, unidade, função), patient/prescription last.

---

## 3. Target repository layout

Lives at `/mnt/s/projetos/receituario-modern/` — a **sibling** of `Receituario/`, never inside it.

```
receituario-modern/
├── pom.xml                         # parent (or single-module) Maven build, Java 21
├── docker-compose.yml              # postgres (dev) + adminer; optional app image
├── README.md
├── backend/
│   ├── pom.xml
│   ├── src/main/java/br/gov/mandaguari/saude/
│   │   ├── Application.java
│   │   ├── config/                 # SecurityConfig, JwtConfig, OpenApiConfig, CorsConfig, JpaAuditingConfig
│   │   ├── common/
│   │   │   ├── error/              # GlobalExceptionHandler (RFC-7807 ProblemDetail), domain exceptions
│   │   │   ├── audit/              # @Auditable, PHI-access audit writer (replaces SAU_LOG)
│   │   │   ├── validation/         # @CPF, @CNPJ, @CNS validators (port psau_val_cnpjcpf, PSAU_VER_CNS)
│   │   │   └── persistence/        # BaseEntity, naming strategy preserving GeneXus column names
│   │   ├── security/               # auth controller, JwtService, UserDetails over SAU_USU/SYS_PES
│   │   └── <slice>/                # ONE package per migrated domain, e.g. paciente/
│   │       ├── domain/             #   JPA entity (+ value objects, enums from gxdomain*)
│   │       ├── repository/         #   Spring Data repository
│   │       ├── dto/                #   request/response records
│   │       ├── mapper/             #   MapStruct entity<->dto
│   │       ├── service/            #   business rules mined from <trn>_impl.java
│   │       └── web/                #   @RestController
│   ├── src/main/resources/
│   │   ├── application.yml         # profiles: local, test, parity
│   │   └── db/migration/           # Flyway: V1__baseline.sql + forward migrations
│   └── src/test/java/br/gov/mandaguari/saude/
│       ├── <slice>/                #   *ServiceTest (JUnit5+Mockito), *ControllerIT (Testcontainers+RestAssured)
│       └── parity/                 #   golden-master tests vs running GeneXus app
├── frontend/
│   ├── package.json                # React 18, TS, Vite, TanStack Query, RHF+Zod, React Router, Tailwind
│   ├── orval.config.ts             # generate TS client from backend OpenAPI
│   └── src/
│       ├── lib/                    # api client, query client, auth context, masks (CPF/CNS/CEP)
│       ├── components/             # shadcn/ui primitives, shared form/table widgets
│       └── features/<slice>/       # list (replaces hww), detail (hview), form (transaction), lookup (hprompt)
└── .github/workflows/ci.yml        # build+test backend & frontend
```

**Backend package convention:** `br.gov.mandaguari.saude.<slice>` where `<slice>` is the lowercase
domain noun (e.g. `paciente`, `remedio`, `especialidade`) — **not** the GeneXus table name. We
translate GeneXus Portuguese-abbreviated names to readable domain names (see Naming Map, §7).

---

## 4. The Slice Contract (`SLICE-SPEC`)

The central data structure of the factory. `gx-extractor` + `gx-rule-miner` produce it; every
generator agent consumes it. One file per slice at `.planning/migration/slices/<TRN>.slice.md`
with a fenced ```yaml front-block + prose rule descriptions.

```yaml
slice: SAU_PAC                 # GeneXus transaction name (uppercase)
domain: paciente               # target package / feature name (lowercase)
description: Paciente          # from .java header
wave: 7                        # from BACKLOG.md
complexity: XL
primary_table: SAU_PAC
constellation:                 # which GeneXus objects make up this slice
  transaction: sau_pac.java
  impl: sau_pac_impl.java      # 947 KB — rule source
  bc: sau_pac_bc.java
  workwith: hwwsau_pac.java    # → list/grid screen
  view: hviewsau_pac.java      # → read-only detail
  prompts: [hpromptsau_pac.java, hpromptsau_pac2.java]   # → lookups
  sections: [hsau_pacgeneral.java]
  sdt: SdtSAU_PAC.java
depends_on: [SAU_UNI, SAU_BAI, SAU_CEP, SYS_PES, SAU_TIPLOG, ...]  # must be migrated first
fields:                        # from gxmetadata/<trn>.json if BC, else DB introspection
  - { gx: PacPesCod,  name: codigo,        type: int,    pk: true,  nullable: false, domain: Codigo14 }
  - { gx: PacPesNom,  name: nome,          type: string, len: 50,   nullable: true,  picture: "@!" }
  - { gx: PacPesCPFCNPJ, name: cpfCnpj,    type: string, len: 14,   validator: CPF }
  # ...
keys: { pk: [PacPesCod], unique: [...], fks: [{ cols: [PacUniCod], ref: SAU_UNI }] }
endpoints:                     # target REST surface
  - { method: GET,    path: /api/pacientes,        from: hwwsau_pac,  desc: list/search }
  - { method: GET,    path: /api/pacientes/{id},   from: hviewsau_pac }
  - { method: POST,   path: /api/pacientes,        from: sau_pac INS }
  - { method: PUT,    path: /api/pacientes/{id},   from: sau_pac UPD }
  - { method: DELETE, path: /api/pacientes/{id},   from: sau_pac DLT }
  - { method: GET,    path: /api/pacientes/lookup, from: hpromptsau_pac, desc: autocomplete }
rules:                         # mined business rules — each becomes a service method + unit test
  - id: R1
    when: insert|update
    desc: "CPF must be a valid Brazilian CPF unless 'permitir cadastro sem CPF' is set"
    source: "sau_pac_impl.java:<line>, psau_val_cnpjcpf.java"
    test: PacienteServiceTest#rejectsInvalidCpf
  - id: R2
    desc: "PacPesTip is read-only / derived"
    source: gxmetadata/sau_pac.json (ReadOnly:true)
  # ...
phi_fields: [nome, cpfCnpj, cns, endereco, ...]   # require audit + log redaction (LGPD)
auth: { roles_required: [SAUDE_CADASTRO], notes: "verify against acessamodulo.java + SAU_USU perms" }
parity:                        # golden-master capture plan
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios: [list default page, view by id, insert valid, insert invalid CPF, ...]
status: extracted              # extracted|specced|backend|frontend|tested|verified|cutover
open_questions:                # things to confirm with KB/IDE or regulatory lead
  - "Password hashing scheme in SAU_USU?"
```

**Why this shape:** it is the lossless interface between "understanding GeneXus" and "writing
Spring/React." Rules carry their **source line citation** (auditability) and their **target test
name** (every rule is verifiable). PHI fields drive audit + redaction automatically.

---

## 5. The pipeline (which agent does what)

```
 gx-inventory ──► BACKLOG.md (once)         scaffold-target ──► receituario-modern/ skeleton (once)
                                                     │
 per slice:                                          ▼
 ┌────────────┐   ┌──────────────┐   ┌──────────────────┐
 │gx-extractor│──►│ gx-rule-miner│──►│ gx-schema-mapper │──►  SLICE-SPEC complete
 └────────────┘   └──────────────┘   └──────────────────┘
        │ reads constellation,  │ deep-reads _impl.java   │ introspects live PG /
        │ metadata, builds spec │ action ladders→rules    │ gxmetadata → entity+Flyway
                                                     │
        ┌────────────────────────────┬───────────────┴──────────────┐
        ▼                            ▼                               ▼
 ┌────────────────────┐    ┌──────────────────┐          ┌──────────────────┐
 │springboot-generator│    │ react-generator  │          │   test-author    │
 │ entity/repo/dto/   │    │ list/detail/form/│          │ unit + IT +      │
 │ mapper/service/web │    │ lookup + client  │          │ parity tests     │
 └─────────┬──────────┘    └──────────────────┘          └────────┬─────────┘
           └───────────────────────┬─────────────────────────────┘
                                    ▼
                        ┌────────────────────┐     ┌──────────────────┐
                        │  parity-verifier   │────►│ migration-reviewer│
                        │ capture legacy →   │     │ correctness, LGPD/│
                        │ run new → diff     │     │ PHI, parity gaps  │
                        └────────────────────┘     └──────────────────┘
```

Stages can run partly in parallel (backend / frontend / test scaffolding are independent once the
SLICE-SPEC is frozen). `migrate-slice` is the skill that orchestrates the whole chain for one
transaction; `gx-extract` runs just the understanding phase so a human can review the SLICE-SPEC
before any code is generated.

---

## 6. Testing strategy

| Level | Tooling | What it proves |
|-------|---------|----------------|
| **Unit** | JUnit 5 + Mockito + AssertJ | Each mined `rule` (§4) → one test. Service logic in isolation, repository mocked. |
| **Integration** | Testcontainers (PostgreSQL) + RestAssured + `@SpringBootTest` | Full HTTP→service→JPA→DB. Flyway runs the baseline into the container; seed realistic rows (shape from live DB). Validation, security (JWT), error contracts. |
| **Parity (golden master)** | RestAssured against the **running GeneXus app** + recorded fixtures | The new endpoint returns the same business result as legacy for identical inputs. Captured by `parity-verifier`, stored under `src/test/resources/parity/<slice>/`. Run in the `parity` profile. |
| **Frontend** | Vitest + Testing Library; Playwright (optional e2e) | Component behavior; form validation mirrors backend. |

Coverage gate per slice: every `rule` in the SLICE-SPEC has a passing unit test, and every
`endpoint` has an integration test. Parity scenarios are the acceptance criteria for cutover.

---

## 7. Conventions

**Persistence (keep the schema):**
- Entities map to existing tables with `@Table(name="SAU_PAC")` and `@Column(name="PacPesNom")` —
  **preserve GeneXus physical names**; expose clean names only in DTOs via MapStruct.
- No schema changes in Phase 1. Flyway `V1__baseline.sql` is the introspected current schema with
  `baseline-on-migrate: true` so the existing populated DB is adopted, not recreated.
- `gxdomain*.java` enums → Java enums or DB-backed reference where appropriate.

**API:**
- REST, plural resource nouns in Portuguese domain terms (`/api/pacientes`, `/api/remedios`).
- DTOs are Java `record`s. Errors are RFC-7807 `ProblemDetail`. Pagination via Spring `Pageable`.
- OpenAPI is the contract; the frontend client is generated, never hand-written.

**Security:**
- Stateless JWT (access + refresh). `UserDetailsService` reads `SAU_USU`/`SYS_PES`.
  Password verification must match the legacy scheme — confirm hashing during rule-mining
  (open question per the auth slice). Method-level `@PreAuthorize` from `acessamodulo` permissions.
- CORS locked to the frontend origin. Secrets via env/`application-*.yml` not committed.

**LGPD / PHI:**
- `phi_fields` in the SLICE-SPEC are tagged `@Auditable`; reads/writes emit an audit record
  (replacing `SAU_LOG`). A logging filter redacts PHI from logs. Public blob paths forbidden
  (CONCERNS C3) — files go through authenticated, ACL-checked endpoints.

**Naming Map (GeneXus → domain):** maintained in `.planning/migration/NAMING.md`, seeded from the
domain glossary (`pac`→paciente, `pro`→profissional, `rem`→remedio, `esp`→especialidade,
`uni`→unidade, `usu`→usuario, `recesp`→receituario-especial, `fun`→funcionario, `pes`→pessoa).

---

## 8. Factory component index

| Component | Type | Role |
|-----------|------|------|
| `gx-extractor` | agent | Read constellation + metadata → draft SLICE-SPEC (structure, endpoints). |
| `gx-rule-miner` | agent | Deep-read `_impl.java` → business rules with source citations + target test names. |
| `gx-schema-mapper` | agent | Introspect live PG / gxmetadata → JPA entity mapping + Flyway DDL for the slice's tables. |
| `springboot-generator` | agent | Emit entity/repo/dto/mapper/service/controller from SLICE-SPEC. |
| `react-generator` | agent | Emit list/detail/form/lookup feature + wire generated client. |
| `test-author` | agent | Emit unit + integration + parity tests from rules & endpoints. |
| `parity-verifier` | agent | Capture legacy responses, run new endpoints, diff, report gaps. |
| `migration-reviewer` | agent | Review a finished slice: correctness, LGPD/PHI, security, parity coverage. |
| `scaffold-target` | skill | One-time: create the monorepo skeleton, security baseline, Flyway baseline, CI. |
| `gx-inventory` | skill | Build/refresh `BACKLOG.md` (dependency-ordered waves). |
| `gx-extract` | skill | Run extractor+miner+schema-mapper → reviewable SLICE-SPEC (no code yet). |
| `migrate-slice` | skill | Orchestrate the full pipeline for one transaction. |
| `verify-parity` | skill | Run parity tests for a slice against the running GeneXus app. |
| `migration-status` | skill | Dashboard of slice statuses across the backlog. |
| regen-guard | hook | PreToolUse: block writes to `Receituario/JavaModel/web/**` and the Tomcat deploy tree. |
| secret-guard | hook | PreToolUse: block committing/writing DB creds, JWT secrets, PHI fixtures. |

---

## 9. How to run a migration

```
/scaffold-target                 # once — create receituario-modern/ + baseline
/gx-inventory                    # once — (re)build BACKLOG.md
/gx-extract SAU_ESP              # produce SLICE-SPEC for review (start with a leaf table)
# human reviews .planning/migration/slices/SAU_ESP.slice.md
/migrate-slice SAU_ESP           # generate backend+frontend+tests
/verify-parity SAU_ESP           # golden-master vs legacy
# review, then cut the proxy route over → mark status: cutover
/migration-status                # see progress
```

---

## 10. Prerequisites (this box currently lacks them)

The factory generates code; to *compile & test* it the environment needs: **JDK 21**, **Maven**,
**Docker** (for Testcontainers), **psql/JDBC** (for live introspection). `scaffold-target` writes a
`docker-compose.yml` for Postgres and documents toolchain install (SDKMAN for JDK/Maven). Node 18
+ npm are already present for the frontend.

---

*Living document — update as conventions evolve. Slice specs live in `slices/`. Backlog in `BACKLOG.md`.*
