# SLICE-SPEC — SAU_ESP (Especialidade)

> Reference slice. Fields/types are mined from `sau_esp_impl.java` attribute vars (`A<n>Esp...`) and
> the glossary — **unverified against the live DB**; `gx-schema-mapper` must confirm types/lengths/
> nullability by introspecting `saude-mandaguari` before this is marked beyond `specced`.

```yaml
slice: SAU_ESP
domain: especialidade
description: Especialidades
wave: 2
complexity: M           # sau_esp_impl.java ~232 KB
primary_table: SAU_ESP
constellation:
  transaction: sau_esp.java
  impl: sau_esp_impl.java            # 232 KB — rule source
  bc: null                           # not a Business Component (no gxmetadata json)
  workwith: null                     # no hwwsau_esp in this regen (list via prompt/Work-With pattern not applied)
  view: null
  prompts: [hpromptsau_esp.java]     # FK lookup popup → /api/especialidades/lookup
  sections: []
  sdt: null
depends_on: [SAU_CBOR]               # CBO occupation lookup (EspCborCod → EspCborDes)
# cross-cutting (every object): SAU_LOG (audit), SAU_USU/SAU_USUCON/SAU_PRF/SAU_PRFCON (auth) — handled by common/
fields:
  - { gx: A27EspCod,   name: codigo,     type: int,     pk: true,  nullable: false, domain: Codigo06, note: "PK" }
  - { gx: A33EspNom,   name: nome,       type: string,  len: 50,   nullable: false, picture: "@!" }
  - { gx: A570EspSit,  name: situacao,   type: string,  len: 1,    nullable: true,  note: "status flag e.g. A/I — confirm domain" }
  - { gx: A569EspAux,  name: auxiliar,   type: boolean, nullable: true, note: "confirm semantics in KB" }
  - { gx: A222EspCborCod, name: cborCodigo, type: int,  nullable: true,  fk: { ref: SAU_CBOR, display: A423EspCborDes }, note: "occupation FK" }
  # read-only lookup display (not persisted on SAU_ESP; comes from SAU_CBOR)
  - { gx: A423EspCborDes, name: cborDescricao, type: string, readOnly: true, source: SAU_CBOR }
  # scheduling-queue parameters per urgency tier (MuitoUrg, Urg, Pri, Normal):
  - { gx: A571EspLstAgendEstagnadoMuitoUrg, name: agendaEstagnadoMuitoUrgente, type: int, nullable: true }
  - { gx: A572EspLstAgendEstagnadoNormal,   name: agendaEstagnadoNormal,       type: int, nullable: true }
  - { gx: A573EspLstAgendEstagnadoPri,      name: agendaEstagnadoPrioritario,  type: int, nullable: true }
  - { gx: A574EspLstAgendEstagnadoUrg,      name: agendaEstagnadoUrgente,      type: int, nullable: true }
  - { gx: A575EspLstAgendTempoMaxMuitoUrg,  name: agendaTempoMaxMuitoUrgente,  type: int, nullable: true }
  - { gx: A576EspLstAgendTempoMaxNormal,    name: agendaTempoMaxNormal,        type: int, nullable: true }
  - { gx: A577EspLstAgendTempoMaxPri,       name: agendaTempoMaxPrioritario,   type: int, nullable: true }
  - { gx: A578EspLstAgendTempoMaxUrg,       name: agendaTempoMaxUrgente,       type: int, nullable: true }
  - { gx: A579EspLstAgendVagaMuitoUrgMax,   name: agendaVagaMuitoUrgenteMax,   type: int, nullable: true }
  - { gx: A580EspLstAgendVagaMuitoUrgMin,   name: agendaVagaMuitoUrgenteMin,   type: int, nullable: true }
  - { gx: A581EspLstAgendVagaNorMax,        name: agendaVagaNormalMax,         type: int, nullable: true }
  - { gx: A582EspLstAgendVagaNorMin,        name: agendaVagaNormalMin,         type: int, nullable: true }
  - { gx: A583EspLstAgendVagaPriMax,        name: agendaVagaPrioritarioMax,    type: int, nullable: true }
  - { gx: A584EspLstAgendVagaPriMin,        name: agendaVagaPrioritarioMin,    type: int, nullable: true }
  - { gx: A585EspLstAgendVagaUrgMax,        name: agendaVagaUrgenteMax,        type: int, nullable: true }
  - { gx: A586EspLstAgendVagaUrgMin,        name: agendaVagaUrgenteMin,        type: int, nullable: true }
keys: { pk: [A27EspCod], fks: [{ cols: [A222EspCborCod], ref: SAU_CBOR }] }
endpoints:
  - { method: GET,    path: /api/especialidades,        from: list/search, role: SAUDE_CADASTRO }
  - { method: GET,    path: /api/especialidades/{id},   from: view }
  - { method: POST,   path: /api/especialidades,        from: sau_esp INS, role: SAUDE_CADASTRO }
  - { method: PUT,    path: /api/especialidades/{id},   from: sau_esp UPD, role: SAUDE_CADASTRO }
  - { method: DELETE, path: /api/especialidades/{id},   from: sau_esp DLT, role: SAUDE_CADASTRO }
  - { method: GET,    path: /api/especialidades/lookup, from: hpromptsau_esp, desc: autocomplete }
phi_fields: []          # specialty catalog — no patient PHI
auth: { roles_required: [SAUDE_CADASTRO], notes: "confirm exact permission in acessamodulo + SAU_PRFCON" }
parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios:
    - list default
    - get by id (existing)
    - get by id (404)
    - insert valid (new code+name+cbor)
    - insert duplicate code (reject)
    - insert missing name (reject)
    - update name + agenda params
    - delete unused; delete referenced-by-profissional (reject — see R4)
status: verified       # unit 10/10 + IT 13/13 + frontend 5/5; parity 9/9 (2026-06-22, after schema-mismatch fix)
# parity_result: 9/9 PARITY vs shared snapshot (DB-state). Report: backend/src/test/resources/parity/especialidade/PARITY-REPORT.md.
# BLOCKER FOUND & FIXED — entity types did not match the live DB (Testcontainers passed only because V1 declared
# the same wrong types). create/update returned 500 (PSQLException 42804). Fixed:
#   - auxiliar Boolean → EspAux INTEGER     : @Convert(BooleanToIntegerConverter) (0/1)
#   - situacao String  → EspSit SMALLINT(1/2): @Convert(SituacaoToShortConverter)
#   - 16 × agenda* Integer → SMALLINT        : @JdbcTypeCode(Types.SMALLINT)
#   - V7__sau_esp_type_alignment.sql aligns the Flyway/Testcontainers schema (idempotent; no-op on live).
# Full mvn verify green (173 unit / 246 IT). ⚠ LESSON: pre-introspection "tested" leaf slices
# (SAU_DIS/SAU_LOC/SAU_BAI/SAU_CONCLA/SAU_UNISETOR) likely have the same class of entity-vs-live type mismatch —
# parity is what catches them; verify each before cutover.
```

## Mined rules (gx-rule-miner output — confidence noted; verify against impl line refs)

- **R1** — *required* `nome` on insert/update. `kind: validation` · `confidence: high` ·
  `target: EspecialidadeServiceTest#rejectsBlankNome`. (GeneXus required-attribute on EspNom.)
- **R2** — `codigo` is the primary key; on INSERT it must be unique (reject duplicate). On UPD the key
  is immutable. `kind: validation` · `confidence: high` · `target: #rejectsDuplicateCodigo`,
  `#codigoImmutableOnUpdate`.
- **R3** — `cborCodigo`, when provided, must reference an existing `SAU_CBOR`; `cborDescricao` is
  derived read-only from that lookup (never written by this slice). `kind: derivation+referential` ·
  `confidence: medium` · `target: #rejectsUnknownCbor`, `#derivesCborDescricao`.
- **R4** — DELETE must be blocked if the especialidade is referenced by a profissional
  (`SAU_PROESP`) or appears in `SAU_USUUNI` scope. `kind: referential` · `confidence: medium`
  (inferred from TableAccess listing SAU_PROESP/SAU_USUUNI) · `target: #rejectsDeleteWhenReferenced`.
- **R5** — agenda scheduling params: each `*Min` must be ≤ its `*Max`, and values are non-negative
  integers (minutes/slots). `kind: validation` · `confidence: low` (inferred from field semantics —
  **verify in KB whether the impl enforces this**) · `target: #rejectsVagaMinGreaterThanMax`.
- **R6** — every create/update/delete writes a `SAU_LOG` audit record with the acting `usu_login`.
  `kind: audit` · `confidence: high` · handled by `common/audit`.
- **R7** — `situacao`/`auxiliar` semantics (active flag?) — `confidence: low`, open question.

## Open questions (resolve via KB/IDE before `verified`)
1. Exact PG types/lengths/nullability for every column (live-DB introspection).
2. `EspSit` domain values and `EspAux` meaning.
3. Whether R4 (delete guard) and R5 (min≤max) are actually enforced in `sau_esp_impl.java` or only in
   the DB / not at all — locate the `confirm_0xx` / `checkExtendedTable0xx` handlers.
4. Exact permission name gating especialidade CRUD.
5. **R4 / SAU_USUUNI branch (F1)** — The baseline DDL stub for SAU_USUUNI has no `EspCod` column;
   actual column name linking SAU_USUUNI → SAU_ESP is unknown. Introspect live DB or KB before
   adding `isReferencedByUsuUni()` query in `EspecialidadeRepository`. Low-risk omission given
   medium confidence on this branch of R4.
6. **Parity (F4)** — All 8 parity scenarios in `EspecialidadeParityIT` are `@Disabled` pending
   fixture capture. Run `/verify-parity SAU_ESP` against the live GeneXus instance before marking
   `verified`.
7. **Frontend coverage** — 12/16 agenda scheduling fields absent from the form UI (`EspecialidadeFormPage`
   only shows the 4 vaga fields). Product decision needed: show all 16 or keep current subset.
8. **Frontend delete** — `useDeleteEspecialidade` mutation and delete button are not wired in
   `EspecialidadeListPage`. Backend DELETE endpoint is tested and working.
