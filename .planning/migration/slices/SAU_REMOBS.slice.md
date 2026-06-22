# SLICE-SPEC — SAU_REMOBS (Posologia)

> Wave-1 leaf lookup. Dosage-instructions catalog ("Posologia"). More complex than a two-field
> lookup — has 8 DB columns, auto-assigned PK, and two delete guards (SAU_REMPOSO + SAU_RECESP1).
> Schema is the GeneXus reorg DDL (`sysmar/MDL002/GEN12/SAU_REMOBSConversion.xml`).
> Live-DB introspection NOT run here; `gx-schema-mapper` should confirm against `saude-mandaguari`
> before this advances past `specced`.

```yaml
slice: SAU_REMOBS
domain: posologia
description: Posologia (Dosage Instructions)
wave: 1
complexity: M            # sau_remobs_impl.java ~2709 lines
primary_table: SAU_REMOBS
constellation:
  transaction: sau_remobs.java
  impl: sau_remobs_impl.java          # 2709 lines — rule source
  bc: null
  workwith: null                      # no hwwsau_remobs in regen (managed via the trn itself)
  view: null
  prompts: [hpromptsau_remobs.java]   # FK lookup popup → /api/posologias/lookup
  sections: []
  sdt: null
  procedures:
    - psau_inc_remobs.java             # auto-generates next RemObsCod (MAX+1 strategy)
depends_on: []                        # no outbound FK to any other migrated table
# reverse references (NOT depends_on):
#   SAU_REMPOSO.PosoRemObsCod → SAU_REMOBS.RemObsCod  (delete guard R3) — Wave 3 un-migrated
#   SAU_RECESP1.RemObsCod    → SAU_REMOBS.RemObsCod  (delete guard R4) — Wave 6 un-migrated
# SAU_USU.RemObsUsuCod: RemObsUsuCod references SAU_USU but there is NO FK constraint in DDL;
#   it is set system-side to the current user's integer code (Wave 0 un-migrated → nullable for now)
# cross-cutting: TableAccess lists SAU_LOG (audit) + SAU_USU/SAU_USUCON/SAU_PRF/SAU_PRFCON (auth)
fields:
  - { gx: A9RemObsCod,            name: codigo,           type: int,        pk: true,  nullable: false,
      note: "System-assigned PK N(6,0). Auto-assigned via psau_inc_remobs (SELECT MAX+1). The form
             hides this field (Visible=0 in Start event) — user NEVER enters it. POST body must not
             include it; backend computes it. src Conversion.xml AttriId 9, psau_inc_remobs.java:54-71" }
  - { gx: A154RemObsDes,          name: descricao,        type: string, len: 60, nullable: false,
      note: "DB column VARCHAR(60) AllowNulls=Yes, BUT the transaction REQUIRES it (R2).
             'Informe a observação do medicamento!' src checkExtendedTable0G12:908-914.
             Form label 'Descrição'. @NotBlank in DTO. src Conversion.xml AttriId 154" }
  - { gx: A1020RemObsInternamento, name: internamento,    type: boolean,    nullable: true,
      note: "Prescrição do Internamento? (optional). DB BOOLEAN AllowNulls=Yes. Default false.
             Not shown on the SAU_REMOBS form; set from prescription screens. src AttriId 1020" }
  - { gx: A1023RemObsQuantidadeDose, name: quantidadeDose, type: decimal, scale: 2, nullable: true,
      note: "Quantidade de dose, NUMERIC(8,2) nullable. Not on the main form. AttriId 1023" }
  - { gx: A1021RemObsMedidaDose,  name: medidaDose,       type: int,        nullable: true,
      note: "Unidade Medida da dose, integer nullable. No FK constraint in DDL — treat as plain int.
             Open question: is this an FK to a unit-of-measure table? src AttriId 1021" }
  - { gx: A1022RemObsIntervaloHoras, name: intervaloHoras, type: int,       nullable: true,
      note: "Intervalo entre doses em horas. DB smallint, N(2,0) → 0..99 range. Not on main form.
             src AttriId 1022" }
  - { gx: A1019RemObsDuracaoDias, name: duracaoDias,      type: int,        nullable: true,
      note: "Por quantos dias o tratamento deve durar. DB smallint, N(3,0) → 0..999 range.
             Not on main form. src AttriId 1019" }
  - { gx: A1018RemObsUsuCod,      name: usuarioCodigo,    type: int,        nullable: true,
      note: "FK (soft, no DDL constraint) to SAU_USU.UsuCod. System-set on INSERT to the current
             user's integer code (afterConfirm0G12:1550-1555). NOT user-editable. Leave nullable
             in Wave 1 since SAU_USU is un-migrated; revisit when Wave 0 auth is done. AttriId 1018" }
keys:
  pk: [A9RemObsCod]
  unique: [{ index: ISAU_REMOBS, cols: [A9RemObsCod] }]
  indexes: [{ index: USAU_REMOBS_DESC, cols: [A9RemObsCod DESC] }]
  fks: []                              # no outbound FK in DDL
ddl_baseline: |
  CREATE TABLE SAU_REMOBS (
    RemObsCod             integer       NOT NULL,
    RemObsDes             VARCHAR(60),
    RemObsInternamento    BOOLEAN,
    RemObsQuantidadeDose  NUMERIC(8,2),
    RemObsMedidaDose      integer,
    RemObsIntervaloHoras  smallint,
    RemObsDuracaoDias     smallint,
    RemObsUsuCod          integer,
    PRIMARY KEY(RemObsCod)
  );
  CREATE INDEX USAU_REMOBS_DESC ON SAU_REMOBS (RemObsCod DESC);
endpoints:
  - { method: GET,    path: /api/posologias,           from: list/search,      role: SAUDE_CADASTRO }
  - { method: GET,    path: /api/posologias/{id},      from: view }
  - { method: POST,   path: /api/posologias,           from: sau_remobs INS,   role: SAUDE_CADASTRO,
      note: "Body: {descricao: string, ...optional fields}. Backend auto-assigns codigo (MAX+1)." }
  - { method: PUT,    path: /api/posologias/{id},      from: sau_remobs UPD,   role: SAUDE_CADASTRO }
  - { method: DELETE, path: /api/posologias/{id},      from: sau_remobs DLT,   role: SAUDE_CADASTRO }
  - { method: GET,    path: /api/posologias/lookup,    from: hpromptsau_remobs, desc: autocomplete,
      note: "Accepts ?descricao= (ILIKE) and optional ?posologiaIndividual=1 (filter by current user's
             RemObsUsuCod). Returns [{codigo, descricao, usuarioCodigo}]." }
phi_fields: []           # dosage-instructions catalog — no patient PHI
auth:
  roles_required: [SAUDE_CADASTRO]
  notes: >
    Legacy disables integrated security (sau_remobs_impl.java:2009 IntegratedSecurityEnabled() → false).
    Gate CRUD behind SAUDE_CADASTRO to match reference slices; confirm exact permission.
parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios:
    - list default
    - get by id (existing)
    - get by id (404)
    - insert valid (descricao only, codigo auto-assigned)
    - insert missing descricao (reject — R2)
    - update descricao
    - update optional fields (internamento, quantidadeDose, etc.)
    - delete unused (ok)
    - delete referenced by SAU_REMPOSO (reject — R3)
    - delete referenced by SAU_RECESP1 (reject — R4)
    - lookup by descricao (partial match)
status: verified       # unit 8/8 + IT 12/12; parity 11/11 (2026-06-21)
# parity_result: 11/11 PARITY vs shared snapshot (DB-state). Report: backend/src/test/resources/parity/posologia/PARITY-REPORT.md.
#   #5 rejects via 400 (Bean Validation @NotBlank) not 422 — behavior matches legacy; same status nuance as RF6. No business divergences.
#   PK generated MAX+1; delete-guards R3 (SAU_REMPOSO) + R4 (SAU_RECESP1) verified. Synthetic rows (ZZPOSO) cleaned up.
```

## Mined rules (from sau_remobs_impl.java — confidence noted; cite line refs)

- **R1** — `codigo` is the PK, **system-assigned** (NOT user-entered). The form hides the code
  field (`edtRemObsCod_Visible = 0` in `e110G2:768-769`). The `psau_inc_remobs` procedure computes
  the next code as `SELECT MAX(RemObsCod)+1 FROM SAU_REMOBS` (or 1 if empty).
  `kind: generation` · `confidence: high` · src `psau_inc_remobs.java:54-71`,
  `afterConfirm0G12:1540-1549` · `target: PosologiaServiceTest#autoAssignsCodigoOnInsert`.
  **Impl note:** Use `SELECT MAX(RemObsCod) FROM SAU_REMOBS` in a `@Transactional` service method;
  the PK unique constraint handles race conditions.

- **R2** — `descricao` is **required** by the transaction: empty → *"Informe a observação do
  medicamento!"*. NOTE the DB column is `VARCHAR(60)` nullable, but the transaction enforces
  non-empty — the new DTO marks it `@NotBlank`. `kind: validation` · `confidence: high` ·
  src `checkExtendedTable0G12:908-914` · `target: #rejectsBlankDescricao`.

- **R3** — **DELETE is blocked when referenced by `SAU_REMPOSO`** (Posologia de Medicamento):
  cursor T000G11 at `onDeleteControls0G12:1417-1424`:
  `SELECT RemCod, PosoRemObsCod FROM SAU_REMPOSO WHERE PosoRemObsCod = ? LIMIT 1`.
  Error: `GXM_del("Posologia de Medicamento")` / `CannotDeleteReferencedRecord`.
  `kind: referential` · `confidence: high` · src `onDeleteControls0G12:1417-1424` ·
  `target: #rejectsDeleteWhenReferencedByRemposo`.
  **Impl note:** `SAU_REMPOSO` is Wave 3 un-migrated → implement as native
  `SELECT count(*)>0 FROM SAU_REMPOSO WHERE PosoRemObsCod=?` until SAU_REMPOSO has an entity.

- **R4** — **DELETE is blocked when referenced by `SAU_RECESP1`** (Itens do Receituário Controle
  Especial): cursor T000G12 at `onDeleteControls0G12:1425-1432`:
  `SELECT RecEspUniCod, RecEspCod, RecEspSeq FROM SAU_RECESP1 WHERE RemObsCod = ? LIMIT 1`.
  Error: `GXM_del("Itens do Receituário Controle Especial")` / `CannotDeleteReferencedRecord`.
  `kind: referential` · `confidence: high` · src `onDeleteControls0G12:1425-1432` ·
  `target: #rejectsDeleteWhenReferencedByRecesp1`.
  **Impl note:** `SAU_RECESP1` is Wave 6 / **Portaria 344/98 controlled substances** — un-migrated
  → implement as native `SELECT count(*)>0 FROM SAU_RECESP1 WHERE RemObsCod=?` until migrated.

- **R5** — On INSERT, `RemObsUsuCod` is **auto-set to the current user's integer code**
  (`AV7Context.getgxTv_SdtContext_Usucod()`). This is NOT user-editable via the form. In Wave 1
  the JWT does not carry the user's `UsuCod` integer → set `RemObsUsuCod` = null; fill in properly
  when the Wave-0 `SAU_USU` slice is migrated and the JWT claim is added.
  `kind: system-assignment` · `confidence: high` · src `afterConfirm0G12:1550-1555`.

- **R6** — All CUD operations are **audited** via SAU_LOG (`psau_inc_log` called at
  `endLevel0G12:1465-1478`). Map to `AuditService.record("CREATE|UPDATE|DELETE","SAU_REMOBS",codigo)`.
  `kind: audit` · `confidence: high` (explicit `psau_inc_log` call, same pattern as SAU_LOC) ·
  `target: #writesAuditOnCreate`.

- **R7** — Lookup supports **"Posologia Individual"** mode: `hpromptsau_remobs` accepts
  `AV11PosologiaIndividual=1` and adds `WHERE RemObsUsuCod = ?` to the query, filtering to only
  records created by the current user. Expose via `?posologiaIndividual=1` query param in the
  `/lookup` endpoint, using the JWT user's `UsuCod` (or skip in Wave 1, no-op until SAU_USU migrated).
  `kind: filter` · `confidence: high` · src `hpromptsau_remobs_impl.java:1907-1921`.

## Open questions (resolve via KB/IDE / DB before `verified`)
1. **Auth** — legacy disables integrated security; confirm `SAUDE_CADASTRO` is the correct role.
2. **`medidaDose` FK** — `RemObsMedidaDose` looks like a unit-of-measure code. Confirm whether
   it references another table (and which Wave) or is truly a plain integer.
3. **System-assigned code concurrency** — the `MAX()+1` strategy has a race window. Confirm whether
   the legacy app ever ran concurrent inserts, and whether the PK unique-constraint retry is acceptable.
4. **`usuarioCodigo` in Wave 1** — leaving `RemObsUsuCod` null is acceptable since SAU_USU is
   un-migrated. Confirm no Wave-1 query filters by this column.
5. **Live-DB confirmation** — verify PG types (integer PK, BOOLEAN, NUMERIC(8,2), smallints) match
   the reorg DDL; introspection not run here.
6. **Flyway baseline** — need minimal stubs for `SAU_REMPOSO(RemCod, PosoRemObsCod)` and
   `SAU_RECESP1(RecEspUniCod, RecEspCod, RecEspSeq, RemObsCod)` so the delete-guard ITs compile.
