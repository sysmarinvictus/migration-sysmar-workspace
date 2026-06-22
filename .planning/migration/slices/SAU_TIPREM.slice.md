# SLICE-SPEC — SAU_TIPREM (Tipos de Medicamento)

> Wave-1 leaf lookup. Medication-type catalog. Referenced by `SAU_REM` (medicamento). Schema is the
> GeneXus reorg DDL (`sysmar/MDL002/GEN12/SAU_TIPREMConversion.xml`). Live-DB introspection NOT run
> here; `gx-schema-mapper` should confirm against `saude-mandaguari` before this advances past
> `specced`. Combines the SAU_CONCLA pattern (delete-guard vs an un-migrated referencing table) with
> the SAU_LOC pattern (a DB-nullable column the transaction nevertheless requires) + audit.

```yaml
slice: SAU_TIPREM
domain: tipo-medicamento
description: Tipos de Medicamento
wave: 1
complexity: S            # sau_tiprem_impl.java ~86 KB
primary_table: SAU_TIPREM
constellation:
  transaction: sau_tiprem.java
  impl: sau_tiprem_impl.java          # ~86 KB — rule source
  bc: null                            # not a Business Component
  workwith: null                      # no hwwsau_tiprem in this regen
  view: null
  prompts: [hpromptsau_tiprem.java]   # FK lookup popup → /api/tipos-medicamento/lookup
  sections: []
  sdt: null
depends_on: []                        # owns one table; no outbound FK to migrate first
# reverse reference (NOT a depends_on): SAU_REM.TipRemCod → SAU_TIPREM.TipRemCod drives the delete
#   guard (R3). SAU_REM is Wave 3 (un-migrated) → implement as a native count against SAU_REM until
#   SAU_REM has an entity, then swap to a derived count. (TipRemCod per Conversion.xml TakesValueFrom SAU_REM.)
# cross-cutting: TableAccess lists SAU_LOG (audit) + SAU_USU/SAU_USUCON/SAU_PRF/SAU_PRFCON (auth) —
#   handled by common/. This transaction IS audited (SAU_LOG present), like SAU_LOC.
fields:
  - { gx: A252TipRemCod, name: codigo, type: int, pk: true, nullable: false,
      note: "user-entered N(6,0) → integer, range 0..999999 (Autonumber=No, Autogenerate=No). src Conversion.xml AttriId 252" }
  - { gx: A1004TipRemDes, name: descricao, type: string, len: 50, nullable: false,
      note: "DB column VARCHAR(50) AllowNulls=Yes, BUT the transaction REQUIRES it (R2). src Conversion.xml AttriId 1004, sau_tiprem_impl.java:777-780" }
keys:
  pk: [A252TipRemCod]
  unique: [{ index: ISAU_TIPREM, cols: [A252TipRemCod] }]
  indexes: [{ index: USAU_TIPREM_DESC, cols: [A252TipRemCod DESC] }]
  fks: []                              # no outbound FK
ddl_baseline: |
  CREATE TABLE SAU_TIPREM (TipRemCod integer NOT NULL, TipRemDes VARCHAR(50), PRIMARY KEY(TipRemCod));
  CREATE INDEX USAU_TIPREM_DESC ON SAU_TIPREM (TipRemCod DESC);
endpoints:
  - { method: GET,    path: /api/tipos-medicamento,        from: list/search,   role: SAUDE_CADASTRO }
  - { method: GET,    path: /api/tipos-medicamento/{id},   from: view }
  - { method: POST,   path: /api/tipos-medicamento,        from: sau_tiprem INS, role: SAUDE_CADASTRO }
  - { method: PUT,    path: /api/tipos-medicamento/{id},   from: sau_tiprem UPD, role: SAUDE_CADASTRO }
  - { method: DELETE, path: /api/tipos-medicamento/{id},   from: sau_tiprem DLT, role: SAUDE_CADASTRO }
  - { method: GET,    path: /api/tipos-medicamento/lookup, from: hpromptsau_tiprem, desc: autocomplete }
phi_fields: []           # medication-type reference data — no patient PHI
auth:
  roles_required: [SAUDE_CADASTRO]
  notes: >
    Legacy disables integrated security here (sau_tiprem.java:33 IntegratedSecurityEnabled() → false).
    Gate CRUD behind SAUDE_CADASTRO to match the reference slices; confirm the exact permission.
parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios:
    - list default
    - get by id (existing)
    - get by id (404)
    - insert valid (codigo + descricao)
    - insert missing descricao (reject — R2)
    - insert duplicate codigo (reject — R1)
    - insert codigo out of range > 999999 (reject — R1)
    - update descricao
    - delete unused (ok)
    - delete referenced by a medicamento in SAU_REM (reject — R3)
status: verified       # unit 7/7 + IT 12/12 + frontend 3/3; parity 10/10 (2026-06-21)
# parity_result: 10/10 PARITY vs shared snapshot (DB-state). Report: backend/src/test/resources/parity/tipo-medicamento/PARITY-REPORT.md.
#   #5/#7 reject via 400 (Bean Validation) not 422 — behavior matches legacy (rejects); same status nuance as RF6. No business divergences.
#   Verified Flyway-off + ddl-auto=none; client-supplied PK (no sequence). Synthetic rows (ZZTIP) cleaned up.
```

## Migration log (migrate-slice, 2026-06-15)
Generated backend + frontend + tests in `receituario-modern` (commit `d4497fe`), following SAU_CONCLA
(delete-guard) + SAU_LOC (required field + audit).
- Backend `tipomedicamento/`: entity/dto/mapper/repository/service/controller. Rules R1–R4 with
  `// R<n>` citations; delete-guard via native count against the un-migrated SAU_REM.
- Flyway `V1`: added `SAU_TIPREM` + a minimal `SAU_REM` stub for the R3 guard.
- Frontend `features/tipo-medicamento/`: schema/api/list/form/schema-test; routes + nav in `App.tsx`.
- **Built & tested**: `mvn verify` → `TipoMedicamentoServiceTest` 7/7, `TipoMedicamentoControllerIT`
  12/12 (Testcontainers), full suite green; frontend `tsc` clean + 14/14 vitest.

Parity DEFERRED (no reachable legacy / non-prod DB here). `TipoMedicamentoParityIT` stubs ready.

## Mined rules (from sau_tiprem_impl.java — confidence noted; cite line refs)

- **R1** — `codigo` is the PK, a user-entered `integer` (GeneXus N(6,0), range **0..999999**;
  Autonumber=No/Autogenerate=No). Unique on INSERT (`DuplicatePrimaryKey`), immutable on UPDATE —
  standard GeneXus transaction behavior. `kind: validation` · `confidence: high` (uniqueness/
  immutability) / `medium` (exact 0..999999 bound — infer from N(6,0)) · src `Conversion.xml` AttriId
  252, insert cursor · `target: TipoMedicamentoServiceTest#rejectsDuplicateCodigo`,
  `#codigoImmutableOnUpdate`, `#rejectsCodigoOutOfRange`.
- **R2** — `descricao` is **required** by the transaction: empty → *"Informe a Descrição do Tipo de
  Medicamento!"*. NOTE the DB column is `VARCHAR(50)` nullable, but the transaction enforces non-empty
  — the new DTO marks it `@NotBlank`. `kind: validation` · `confidence: high` · src
  `sau_tiprem_impl.java:777-780` · `target: #rejectsBlankDescricao`.
- **R3** — **DELETE is blocked when referenced by a Medicamento** (`SAU_REM`): cursor `T001A11`; if a
  row is found → `GXM_del("Medicamento")` / `CannotDeleteReferencedRecord`. `kind: referential` ·
  `confidence: high` · src `onDeleteControls :1225-1232` · `target: #rejectsDeleteWhenReferencedByMedicamento`.
  **Impl note:** referencing column is `SAU_REM.TipRemCod` (Conversion.xml TakesValueFrom SAU_REM).
  SAU_REM is **Wave 3 / un-migrated** → implement the guard as a native
  `SELECT count(*)>0 FROM SAU_REM WHERE TipRemCod=?` until SAU_REM has an entity.
- **R4** — create/update/delete are **audited** (TableAccess lists `SAU_LOG`, like SAU_LOC). Map to
  `AuditService.record("CREATE|UPDATE|DELETE", "SAU_TIPREM", codigo)`. `kind: audit` ·
  `confidence: medium` (TableAccess inference — GeneXus audit is declarative) · `target: #writesAuditOnCreate`.

## Open questions (resolve via KB/IDE / DB before `verified`)
1. **Auth** — legacy disables security here; confirm the role (`SAUDE_CADASTRO` proposed) gating CRUD.
2. **Código range** — confirm the transaction's exact `TipRemCod` bound (assumed 0..999999 from N(6,0)).
3. **Live-DB confirmation** — verify PG types/nullability match the reorg DDL (`integer` PK,
   `VARCHAR(50)` descricao); introspection not run here.
4. **Delete-guard column** — confirm `SAU_REM.TipRemCod` is the (only) referencing column; until
   SAU_REM migrates the guard is a native query (seed a minimal `SAU_REM(RemCod, TipRemCod)` stub for the IT).
