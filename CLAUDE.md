# CLAUDE.md — Receituário migration workspace

This directory holds a **GeneXus → Spring Boot/React migration**. Read this first; it points to the
authoritative docs rather than repeating them.

## What's here

| Path | What it is | Editable? |
|------|-----------|-----------|
| `Receituario/JavaModel/web/` | **Legacy GeneXus Java regen output** (the system being replaced) | **READ-ONLY** — see Rule 1 |
| `receituario-modern/` | **New app** — Spring Boot 3 (Java 21) backend + React/TS frontend | yes |
| `.planning/migration/` | Migration plan, contracts, per-slice specs | yes |
| `.planning/codebase/` | Map of the legacy system (STACK/ARCHITECTURE/CONCERNS/…) | yes |
| `.claude/agents`, `.claude/skills` | The **migration factory** (agents + slash-command skills) | yes |
| `.claude/hooks/guard.mjs`, `.claude/settings.json` | PreToolUse guard hook | yes |

## Strategy (decided — don't relitigate without asking)
- **Strangler-fig, slice-by-slice.** Migrate one transaction-domain at a time, dependency-ordered
  (leaf lookups first, `SAU_PAC`/`SAU_RECESP` last). Both apps run against the **same** PostgreSQL
  DB during transition.
- **Keep the existing `saude-mandaguari` schema** (introspect it). No schema redesign in Phase 1.
- Stack: Java 21 / Spring Boot 3.4 / Spring Data JPA / Flyway / Spring Security + JWT / springdoc /
  MapStruct; React 18 + TS + Vite / TanStack Query / RHF + Zod / Tailwind; JUnit 5 + Mockito,
  Testcontainers + RestAssured, golden-master parity.

## Critical rules (non-negotiable)
1. **Never edit `Receituario/JavaModel/web/**` or any `ReceituarioJavaEnvironment` tree.** It is
   GeneXus-regenerated; edits are wiped on the next Build All. The regen-guard hook blocks such
   writes. Real legacy changes go in the GeneXus KB; all new code goes in `receituario-modern/`.
2. **Preserve physical DB names.** Entities pin `@Table`/`@Column` to GeneXus names; clean names
   appear only in DTOs/UI. `hibernate.ddl-auto=validate` — Hibernate never alters the schema; Flyway
   owns DDL. (`V1__baseline.sql` is currently a *partial* baseline — complete it from live-DB intro-
   spection before broad rollout.)
3. **No secrets in the repo.** DB/JWT secrets come from env vars; the secret-scan hook blocks
   committing them. Never propagate the legacy `client.cfg` credentials.
4. **LGPD / PHI.** Audit every PHI read/write (`common/audit`, replaces `SAU_LOG`); never log PHI;
   no public/unauthenticated path returns PHI; no real PHI in committed test fixtures.
5. **`SAU_RECESP`** (controlled substances, Portaria 344/98) needs regulatory sign-off before cutover.
6. **Don't fabricate business rules or claim un-run tests pass.** Mined rules cite `file:line` and
   carry a confidence level; low-confidence rules need KB/IDE verification.

## The migration workflow (factory commands)
```
/migration-status              # backlog + recommended next slice
/scaffold-target               # ONE-TIME: create receituario-modern/ skeleton (already done)
/gx-inventory                  # (re)build .planning/migration/BACKLOG.md
/gx-extract  <SAU_XXX>         # produce a reviewable SLICE-SPEC (no code yet) — review it
/migrate-slice <SAU_XXX>       # generate backend + frontend + tests from the reviewed spec
/verify-parity <SAU_XXX>       # golden-master vs the running GeneXus app, then cut the route over
```
The **SLICE-SPEC** (`.planning/migration/slices/<TRN>.slice.md`) is the contract passed between
stages — its schema is defined in `.planning/migration/ARCHITECTURE.md §4`.

## Reference slice
**Especialidade (`SAU_ESP`)** is implemented end-to-end as the pattern to copy:
`receituario-modern/backend/.../especialidade/` + its tests + `frontend/src/features/especialidade/`.

## Build & test (`receituario-modern/`)
```bash
docker compose up -d postgres                       # local dev DB
cd backend && mvn verify                            # unit + Testcontainers integration tests
mvn -Pparity test                                   # golden-master parity (legacy app must be reachable)
cd ../frontend && npm install && npm run gen:api && npm run dev
```
Toolchain needed: **JDK 21, Maven, Docker, Node 20+** (SDKMAN install commands in
`receituario-modern/README.md`). Subagents in this environment are write-denied — orchestrating
skills persist generated files from the main thread.

## Deeper docs
- `.planning/migration/ARCHITECTURE.md` — full architecture, SLICE-SPEC contract, pipeline, conventions
- `.planning/migration/BACKLOG.md` — dependency-ordered waves · `NAMING.md` — GeneXus→domain map
- `.planning/codebase/CONCERNS.md` — legacy security/LGPD findings to carry forward
- `receituario-modern/README.md` — run/build/test details
