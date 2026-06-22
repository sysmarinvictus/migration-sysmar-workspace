---
name: migrate-slice
description: Orchestrate the full migration of one GeneXus transaction into the new Spring Boot + React app — generate backend, frontend, and tests from a reviewed SLICE-SPEC, then review. Run after gx-extract has produced and you have reviewed the SLICE-SPEC.
---

# migrate-slice <TRANSACTION>

Drives the code-generation half of the pipeline for one slice. Requires a reviewed SLICE-SPEC
(`/mnt/s/projetos/.planning/migration/slices/<TRN>.slice.md`, `status: specced` or later) and a
scaffolded target (`receituario-modern/`).

## Usage
`/migrate-slice SAU_ESP`

## Preconditions (check first)
1. `receituario-modern/` exists — if not, run `/scaffold-target`.
2. The SLICE-SPEC exists and is reviewed — if not, run `/gx-extract <TRN>` first and stop for review.
3. All `depends_on` slices that should precede this one are present (warn about raw-id FKs otherwise).

## Steps
1. Read the SLICE-SPEC and the reference slice (`paciente/`) for the established patterns.
2. **Backend** — spawn `springboot-generator`. Persist its files under
   `backend/src/main/java/br/gov/mandaguari/saude/<domain>/` and any Flyway forward migration.
3. **Tests** — spawn `test-author`. Persist unit + integration + parity-stub tests.
4. **Frontend** — spawn `react-generator`. Persist the feature under `frontend/src/features/<domain>/`
   and register its routes/nav.
   > Steps 2–4 are independent once the spec is frozen and may be dispatched together.
5. **Build & test.** If a JDK 21 + Maven exist: `mvn -q -pl backend test` (Testcontainers needs
   Docker). If Node deps are installed: `npm --prefix frontend run typecheck`. Report results
   honestly; fix compile/type errors. If toolchain is missing, say code is generated-but-unbuilt and
   list the exact commands the user should run.
6. **Review** — spawn `migration-reviewer`. Address BLOCK findings; record FLAGs in the spec's
   open_questions.
7. Update the SLICE-SPEC `status` (`backend`→`tested`→`verified`) and report what's done + remaining.

## Critical constraints
- **Writes that land in `Receituario/JavaModel/web/**` or the Tomcat tree are blocked by the
  regen-guard hook** — all new code goes in `receituario-modern/`. Good.
- Persist generated files from the main thread if subagent writes are denied (collect agent output).
- Don't fabricate rules or claim un-run tests pass. Don't change the physical DB schema.
- For `SAU_RECESP`/auth slices, require the regulatory/security open questions to be resolved (or
  explicitly deferred by the user) before marking `verified`.
