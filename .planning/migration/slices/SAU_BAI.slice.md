# SLICE-SPEC — SAU_BAI (Bairro)

> Wave-1 leaf lookup. Neighborhood catalog. 2-column table; system-assigned PK;
> unique-name rule; two delete guards (SYS_PES + SAU_DIS — both un-migrated Wave-0/2 stubs).
> No FK constraints on SAU_BAI itself — the "bai↔dis cycle" in BACKLOG.md refers only to
> SAU_DIS.DisBaiCod → SAU_BAI (one-way), so SAU_BAI is a pure leaf.

```yaml
slice: SAU_BAI
domain: bairro
description: Bairro (Neighborhood)
wave: 1
complexity: S            # sau_bai_impl.java 2216 lines (standard boilerplate, only 2 data fields)
primary_table: SAU_BAI
constellation:
  transaction: sau_bai.java
  impl: sau_bai_impl.java
  bc: null
  workwith: null
  view: null
  prompts: []
  sections: []
  sdt: null
  procedures:
    - psau_inc_bai.java        # auto-generates next BaiCod (MAX+1 strategy)
    - psau_bai_nome.java       # uniqueness check: SELECT … WHERE UPPER(BaiNom) = UPPER(?) LIMIT 1
    - psau_limpacaracter.java  # strip special chars from BaiNom (normalise to trim() in Wave 1)
depends_on: []              # no outbound FK to any migrated table
# reverse references (NOT depends_on):
#   SYS_PES.PesBaiCod  → SAU_BAI.BaiCod  (delete guard R4) — Wave 0 un-migrated
#   SAU_DIS.DisBaiCod  → SAU_BAI.BaiCod  (delete guard R5) — Wave 2 un-migrated
fields:
  - { gx: A197BaiCod, name: codigo, type: int, pk: true, nullable: false,
      note: "System-assigned PK N(6,0). Auto-assigned via psau_inc_bai (SELECT MAX+1 or 1 if empty).
             Form disables the field (edtBaiCod_Enabled=0 in initialize_properties:1662). POST body
             must not include it. src psau_inc_bai.java:52-70, afterConfirm0313:1401-1409" }
  - { gx: A271BaiNom, name: nome, type: string, len: 50, nullable: false,
      note: "DB column VARCHAR(50) AllowNulls=Yes BUT transaction requires it (R2) AND checks
             uniqueness (R3). Form label 'Nome'. src valid_Bainom:1816-1821, psau_bai_nome__default:122" }
keys:
  pk: [A197BaiCod]
  unique: [{ index: ISAU_BAI, cols: [A197BaiCod] }]
  indexes: [{ index: USAU_BAI_DESC, cols: [A197BaiCod DESC] }]
  fks: []   # no outbound FK in DDL
ddl_baseline: |
  CREATE TABLE SAU_BAI (BaiCod integer NOT NULL, BaiNom VARCHAR(50), PRIMARY KEY(BaiCod));
  CREATE INDEX USAU_BAI_DESC ON SAU_BAI (BaiCod DESC);
endpoints:
  - { method: GET,    path: /api/bairros,       from: list/search,   role: SAUDE_CADASTRO }
  - { method: GET,    path: /api/bairros/{id},  from: view,          role: SAUDE_CADASTRO }
  - { method: POST,   path: /api/bairros,       from: sau_bai INS,   role: SAUDE_CADASTRO,
      note: "Body: {nome: string}. Backend auto-assigns codigo (MAX+1)." }
  - { method: PUT,    path: /api/bairros/{id},  from: sau_bai UPD,   role: SAUDE_CADASTRO }
  - { method: DELETE, path: /api/bairros/{id},  from: sau_bai DLT,   role: SAUDE_CADASTRO }
  - { method: GET,    path: /api/bairros/lookup, from: FK autocomplete, desc: autocomplete }
phi_fields: []           # neighborhood catalog — no patient PHI
auth:
  roles_required: [SAUDE_CADASTRO]
  notes: >
    IntegratedSecurityEnabled() returns false (sau_bai_impl:1850). Gate CRUD behind
    SAUDE_CADASTRO matching reference slices.
parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios:
    - list default
    - get by id (existing)
    - get by id (404)
    - insert valid (nome only, codigo auto-assigned)
    - insert missing nome (reject — R2)
    - insert duplicate nome (reject — R3)
    - update nome
    - update to existing nome (reject — R3)
    - delete unused (ok)
    - delete referenced by SYS_PES (reject — R4)
    - delete referenced by SAU_DIS (reject — R5)
    - lookup by nome (partial match)
status: verified       # parity 12/12 PARITY 2026-06-22. Report: backend/src/test/resources/parity/bairro/PARITY-REPORT.md.
# Entity↔live types match. Auto PK (MAX+1); nome uniqueness (R3→409); both delete-guards live-tested (SAU_DIS R5, SYS_PES R4).
# missing nome rejects via 400 (Bean Validation) not 422 — status nuance (RF6). No business divergences.
```

## Mined rules (from sau_bai_impl.java + psau_bai_nome.java — confidence noted; cite line refs)

- **R1** — `codigo` is the PK, **system-assigned** (NOT user-entered). The form disables the code
  field (`edtBaiCod_Enabled = 0` in `initialize_properties:1662`). The `psau_inc_bai` procedure
  computes the next code as `SELECT MAX(BaiCod)+1` (or 1 if empty).
  `kind: generation` · `confidence: high` · src `psau_inc_bai.java:52-70`,
  `afterConfirm0313:1401-1409`.

- **R2** — `nome` is **required** by the transaction: empty → *"Informe o Nome do Bairro!"*.
  NOTE the DB column is `VARCHAR(50)` nullable, but the transaction enforces non-empty — new DTO
  marks it `@NotBlank`. `kind: validation` · `confidence: high` · src `valid_Bainom:1816-1821`.

- **R3** — `nome` must be **unique** (case-insensitive, trim-normalised): duplicate →
  *"Esse Bairro Já Está Cadastro!"*. The check uses `psau_bai_nome` which runs:
  `SELECT BaiNom, BaiCod FROM SAU_BAI WHERE RTRIM(LTRIM(UPPER(BaiNom))) = RTRIM(LTRIM(UPPER(?)))`
  On UPDATE: exclude current record to allow keeping same name.
  `kind: uniqueness` · `confidence: high` · src `valid_Bainom:1810-1815`,
  `psau_bai_nome__default:122`.

- **R4** — **DELETE is blocked when referenced by `SYS_PES`** (Pessoa): T000311 at
  `onDeleteControls0313:1279-1287`:
  `SELECT PesCod FROM SYS_PES WHERE PesBaiCod = ? LIMIT 1`.
  Error: `GXM_del("Pessoa")` / `CannotDeleteReferencedRecord`.
  `kind: referential` · `confidence: high` · src `onDeleteControls0313:1279-1287`.
  **Impl note:** `SYS_PES` is Wave 0 un-migrated → implement as native
  `SELECT count(*)>0 FROM SYS_PES WHERE PesBaiCod=?` until SYS_PES has an entity.

- **R5** — **DELETE is blocked when referenced by `SAU_DIS`** (Distrito Sanitário): T000312 at
  `onDeleteControls0313:1288-1295`:
  `SELECT DisCod FROM SAU_DIS WHERE DisBaiCod = ? LIMIT 1`.
  Error: `GXM_del("Distrito Sanitário")` / `CannotDeleteReferencedRecord`.
  `kind: referential` · `confidence: high` · src `onDeleteControls0313:1288-1295`.
  **Impl note:** `SAU_DIS` is Wave 2 un-migrated → implement as native
  `SELECT count(*)>0 FROM SAU_DIS WHERE DisBaiCod=?` until SAU_DIS is migrated.

- **R6** — All CUD operations are **audited** via SAU_LOG (`psau_inc_log` called at
  `endLevel0313:1335`). Map to `AuditService.record("CREATE|UPDATE|DELETE","SAU_BAI",codigo)`.
  `kind: audit` · `confidence: high` (explicit `psau_inc_log` call, same pattern as other slices).

## Open questions (resolve via KB/IDE / DB before `verified`)
1. **Auth** — legacy disables integrated security; confirm `SAUDE_CADASTRO` is the correct role.
2. **`psau_limpacaracter`** — strips accented chars or other special characters from BaiNom. Wave 1:
   treat as `trim()`. Confirm whether this normalization matters for the live DB values.
3. **Concurrency** — MAX()+1 strategy. Same as SAU_REMOBS; PK unique-constraint handles races.
4. **Live-DB confirmation** — verify PG column types (integer PK, VARCHAR(50)) match DDL.
5. **Flyway baseline** — need minimal stubs for `SYS_PES(PesCod, PesBaiCod)` and
   `SAU_DIS(DisCod, DisBaiCod)` so the delete-guard ITs compile.
