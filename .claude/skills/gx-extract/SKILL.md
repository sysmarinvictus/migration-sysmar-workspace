---
name: gx-extract
description: Run the understanding phase for one GeneXus transaction — structure extraction, business-rule mining, and schema mapping — to produce a complete, reviewable SLICE-SPEC before any code is generated. Use before migrate-slice so a human can verify the recovered rules.
---

# gx-extract <TRANSACTION>

Produces `/mnt/s/projetos/.planning/migration/slices/<TRN>.slice.md` — the SLICE-SPEC (contract in
ARCHITECTURE.md §4). **Stops before code generation** so the mined rules can be reviewed.

## Usage
`/gx-extract SAU_ESP` (start with a Wave-1 leaf to warm up; the reference target is `SAU_PAC`).

## Steps
1. Read `ARCHITECTURE.md` §4/§7, `NAMING.md`, and the transaction's `BACKLOG.md` row.
2. **Structure** — spawn `gx-extractor` with the transaction name. It returns the structural
   SLICE-SPEC half (fields, keys, constellation, endpoints, depends_on, phi_fields, auth).
3. **Rules** — spawn `gx-rule-miner` with the transaction + the draft spec. It returns the `rules`
   list with source citations, confidence, and target test names.
4. **Schema** — spawn `gx-schema-mapper` with the primary + dependency tables. It returns the entity
   mapping + Flyway plan (introspecting the live DB if reachable, else gxmetadata).
5. **Assemble & persist.** Merge the three outputs into one SLICE-SPEC file and **write it from the
   main thread** to `slices/<TRN>.slice.md` (subagents may be write-denied). Set `status: specced`.
6. **Surface review items.** Print: the rule count by confidence, all `open_questions`, the
   depends_on slices and whether they're migrated yet, and any live-DB-vs-metadata discrepancies.
   Tell the user to review the spec (especially low-confidence rules and PHI tagging) before running
   `/migrate-slice <TRN>`.

## Guardrails
- Do not generate backend/frontend/test code here — extraction only.
- If a `depends_on` table isn't migrated yet, warn that FKs to it will be raw-id columns until then.
- For `SAU_RECESP` (controlled substances) and the auth slice, explicitly list the regulatory/
  security open questions for human sign-off.
