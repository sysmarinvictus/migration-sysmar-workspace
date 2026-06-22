---
name: gx-inventory
description: Build or refresh the dependency-ordered migration backlog (.planning/migration/BACKLOG.md) by analyzing every GeneXus transaction's table dependencies and sorting them into migration waves. Run once at the start and whenever the GeneXus KB is regenerated.
---

# gx-inventory

Produces/refreshes `/mnt/s/projetos/.planning/migration/BACKLOG.md` — the dependency-ordered
roadmap of every transaction grouped into migration waves (leaf reference tables first, patient/
prescription core last).

## Steps
1. Confirm `.planning/migration/ARCHITECTURE.md` exists (the contract). If not, tell the user to set
   up the factory first.
2. Enumerate transactions: `*.java` at `Receituario/JavaModel/web/` that are `sau_*`/`sys_*` and NOT
   `_impl`/`_bc`/`h*`/`p*`/`r*`. For each, read the `Description:` header and `_impl.java` size
   (complexity: S<100KB · M<300 · L<700 · XL≥700).
3. For each transaction, read `Metadata/TableAccess/<Trn>.xml` (case-insensitive match) → the tables
   it touches. Primary table = uppercase of the transaction name; the rest are dependencies.
4. Identify cross-cutting foundation tables (appear in almost every XML — SAU_USU, SAU_USUCON,
   SAU_PRF, SAU_PRFCON, SAU_LOG) → Wave 0 alongside the auth baseline. Identify leaf tables (referenced
   as deps but rarely primary) → early waves.
5. Topologically sort into waves so running top-to-bottom never needs an un-migrated dependency.
   Flag dependency cycles explicitly with a suggested break order.
6. Write `BACKLOG.md` per the format already established (summary table, per-wave tables with
   `Transaction | Primary table | Description | Complexity | Notes/Cycle`, a cycles section, and
   open questions). **Write it from the main thread** — subagents may be write-denied in this env.
7. Report the wave counts and the recommended first slices.

This is read-heavy: you may spawn an `Explore`/`general-purpose` agent to fan out the XML reads and
return structured findings, but **persist `BACKLOG.md` yourself** (the orchestrator writes).
