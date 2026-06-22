# SLICE-SPEC — SAU_LOC (Local)

> Wave-1 leaf lookup. A **Local** (locality/place) belonging to a **Município** (`SYS_MUN`). Schema is
> the GeneXus reorg DDL (`sysmar/MDL002/GEN12/SAU_LOCConversion.xml`, a *new*-table spec → full
> `CREATE TABLE`). Live-DB introspection was NOT run here; `gx-schema-mapper` should confirm against
> `saude-mandaguari` before this advances past `specced`. Closely mirrors the SAU_ESP reference
> pattern (FK to an un-migrated lookup table, with derived read-only display fields + existence check).

```yaml
slice: SAU_LOC
domain: local
description: Local
wave: 1
complexity: S            # sau_loc_impl.java ~102 KB
primary_table: SAU_LOC
constellation:
  transaction: sau_loc.java
  impl: sau_loc_impl.java             # ~102 KB — rule source
  bc: null                            # not a Business Component
  workwith: null                      # no hwwsau_loc in this regen
  view: null
  prompts: [hpromptsau_loc.java]      # FK lookup popup → /api/locais/lookup
  sections: []
  sdt: null
depends_on: [SYS_MUN]                 # município lookup — UN-MIGRATED system table (see note)
# SYS_MUN (Município) is a SYS_* system table, not in the BACKLOG waves. Implement the FK as a raw-id
# column (municipioCodigo) + native queries against SYS_MUN for the existence check and the derived
# display fields, exactly like SAU_ESP's SAU_CBOR handling. Swap to a JPA relationship once/if SYS_MUN
# is migrated.
# cross-cutting: TableAccess lists SAU_LOG (audit) + SAU_USU/SAU_USUCON/SAU_PRF/SAU_PRFCON (auth) —
#   handled by common/ (see R6). Unlike SAU_CONCLA, this transaction IS audited (SAU_LOG present).
fields:
  - { gx: A239LocCod, name: codigo, type: int, pk: true, nullable: false,
      note: "user-entered N(6,0) → integer, range 0..999999 (Autonumber=No, Autogenerate=No). src Conversion.xml AttriId 239" }
  - { gx: A865LocNom, name: nome, type: string, len: 50, nullable: false,
      note: "DB column is VARCHAR(50) AllowNulls=Yes, BUT the transaction REQUIRES it (R2). src Conversion.xml AttriId 865, sau_loc_impl.java:849-852" }
  - { gx: A240LocMunCod, name: municipioCodigo, type: int, nullable: false,
      fk: { ref: SYS_MUN, col: MunCod }, note: "DB column nullable, BUT transaction REQUIRES non-zero (R3) and existing (R4). N(9,0)→integer. src Conversion.xml AttriId 240" }
  # derived read-only (from SYS_MUN, NOT persisted on SAU_LOC):
  - { gx: A862LocMunNom,  name: municipioNome, type: string, readOnly: true, source: "SYS_MUN.MunNom" }
  - { gx: A863LocMunUF,   name: municipioUf,   type: string, readOnly: true, source: "SYS_MUN.MunUF (confirm column name)" }
  - { gx: A864LocMunIBGE, name: municipioIbge, type: string, readOnly: true, source: "SYS_MUN.MunIBGE (confirm column name)" }
keys:
  pk: [A239LocCod]
  unique: [{ index: ISAU_LOC, cols: [A239LocCod] }]
  indexes: [{ index: USAU_LOC_DESC, cols: [A239LocCod DESC] }, { index: ISAU_LOC1, cols: [A240LocMunCod] }]
  fks: [{ cols: [A240LocMunCod], ref: SYS_MUN, refCols: [MunCod] }]
ddl_baseline: |
  CREATE TABLE SAU_LOC (LocCod integer NOT NULL, LocNom VARCHAR(50), LocMunCod integer,
    PRIMARY KEY(LocCod));
  CREATE INDEX USAU_LOC_DESC ON SAU_LOC (LocCod DESC);
  CREATE INDEX ISAU_LOC1 ON SAU_LOC (LocMunCod);
  ALTER TABLE SAU_LOC ADD CONSTRAINT ISAU_LOC1 FOREIGN KEY (LocMunCod) REFERENCES SYS_MUN (MunCod);
  # SAU_LOC is a NEW table in this reorg (full DDL above). SYS_MUN must exist first (its own slice /
  # the partial baseline) for the FK; seed a minimal SYS_MUN stub for the IT like SAU_ESP did for SAU_CBOR.
endpoints:
  - { method: GET,    path: /api/locais,        from: list/search,  role: SAUDE_CADASTRO }
  - { method: GET,    path: /api/locais/{id},   from: view }
  - { method: POST,   path: /api/locais,        from: sau_loc INS,  role: SAUDE_CADASTRO }
  - { method: PUT,    path: /api/locais/{id},   from: sau_loc UPD,  role: SAUDE_CADASTRO }
  - { method: DELETE, path: /api/locais/{id},   from: sau_loc DLT,  role: SAUDE_CADASTRO }
  - { method: GET,    path: /api/locais/lookup, from: hpromptsau_loc, desc: autocomplete }
phi_fields: []           # locality reference data — no patient PHI
auth:
  roles_required: [SAUDE_CADASTRO]
  notes: >
    Legacy disables integrated security here too (sau_loc.java:33 IntegratedSecurityEnabled() → false).
    Gate CRUD behind SAUDE_CADASTRO to match the reference slices; confirm the exact permission.
parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios:
    - list default
    - get by id (existing)
    - get by id (404)
    - insert valid (codigo + nome + municipio existente) — derives municipioNome/Uf/Ibge
    - insert missing nome (reject — R2)
    - insert missing/zero municipio (reject — R3)
    - insert unknown municipio (reject — R4, ForeignKeyNotFound)
    - insert duplicate codigo (reject — R1)
    - insert codigo out of range > 999999 (reject — R1)
    - update nome + municipio
    - delete (allowed — no delete guard, R5)
status: verified       # parity 11/11 PARITY 2026-06-22. Report: backend/src/test/resources/parity/local/PARITY-REPORT.md.
# Entity↔live types match. Client PK; derives municipioNome from SYS_MUN; municipio FK→422; no delete-guard (R5).
# required-field/range failures reject via 400 (Bean Validation) not 422 — status nuance (RF6). No business divergences.
```

## Migration log (migrate-slice, 2026-06-15)
Generated backend + frontend + tests in `receituario-modern` (commit `9734e14`), following SAU_ESP.
- Backend `local/`: entity/dto/mapper/repository/service/controller. Rules R1–R6 implemented with
  `// R<n>` citations; município FK validated + name/UF/IBGE derived via native SYS_MUN projection.
- Flyway `V1`: added `SAU_LOC` + a minimal `SYS_MUN` stub (FK + derived fields).
- Frontend `features/local/`: schema/api/list/form/schema-test; routes + nav in `App.tsx`.
- **Built & tested** (JDK 21 + Maven + Docker in-env): `mvn verify` → `LocalServiceTest` 9/9,
  `LocalControllerIT` 13/13 (Testcontainers), full suite green; frontend `tsc` clean + 11/11 vitest.

Parity DEFERRED (same as SAU_CONCLA — no reachable legacy instance / non-prod DB here).
`LocalParityIT` stubs ready. Status stays `tested` until `/verify-parity SAU_LOC` runs.

## Mined rules (from sau_loc_impl.java — confidence noted; cite line refs)

- **R1** — `codigo` is the PK, a user-entered `integer` (GeneXus N(6,0), range **0..999999**;
  Autonumber=No/Autogenerate=No). Unique on INSERT (`DuplicatePrimaryKey`), immutable on UPDATE —
  standard GeneXus transaction behavior (same shape as SAU_CONCLA R2). `kind: validation` ·
  `confidence: high` (PK uniqueness/immutability) / `medium` (exact 0..999999 bound — infer from
  N(6,0); confirm the `ctol` range check) · src `Conversion.xml` AttriId 239, insert cursor in
  `sau_loc_impl.java` · `target: LocalServiceTest#rejectsDuplicateCodigo`, `#codigoImmutableOnUpdate`,
  `#rejectsCodigoOutOfRange`.
- **R2** — `nome` is **required** by the transaction: empty → *"Informe o Nome do Local!"*. NOTE the
  DB column is `VARCHAR(50)` nullable, but the transaction enforces non-empty — the new DTO marks it
  `@NotBlank`. `kind: validation` · `confidence: high` · src `sau_loc_impl.java:849-852` ·
  `target: #rejectsBlankNome`.
- **R3** — `municipioCodigo` is **required**: zero/absent → *"Informe o Município!"*. DB column is
  nullable but the transaction makes it mandatory. `kind: validation` · `confidence: high` · src
  `sau_loc_impl.java:877-882` · `target: #rejectsMissingMunicipio`.
- **R4** — when `municipioCodigo` is provided it must reference an existing `SYS_MUN`, else
  *"Não existe 'Municipio'."* (`ForeignKeyNotFound`). The município **name/UF/IBGE** are derived
  read-only from `SYS_MUN` (never written by this slice). `kind: referential + derivation` ·
  `confidence: high` · src `checkExtendedTable0B5 :856-872` · `target: #rejectsUnknownMunicipio`,
  `#derivesMunicipioFields`. **Impl note:** SYS_MUN is un-migrated → native existence query
  `SELECT count(*)>0 FROM SYS_MUN WHERE MunCod=?` and a native lookup for `MunNom/MunUF/MunIBGE`.
- **R5** — **no delete guard**: `onDeleteControls0B5` only loads the município display fields; there
  is no referencing-table check, so a Local can be deleted freely. `kind: behavior (negative)` ·
  `confidence: high` · src `sau_loc_impl.java:1361-1379` · `target: #deletesFreely`.
- **R6** — create/update/delete are **audited** (TableAccess lists `SAU_LOG`; unlike SAU_CONCLA).
  Map to `AuditService.record("CREATE|UPDATE|DELETE", "SAU_LOC", codigo)`. `kind: audit` ·
  `confidence: medium` (TableAccess inference — GeneXus audit is declarative, not an explicit write in
  `_impl`) · `target: #writesAuditOnCreate`.

## Open questions (resolve via KB/IDE / DB before `verified`)
1. **SYS_MUN column names** — confirm `MunNom` (confirmed in SYS_MUNConversion.xml), and the exact
   physical names for UF and IBGE (`MunUF` / `MunIBGE` assumed) for the derived-field native query.
2. **Auth** — legacy disables security here; confirm the role (`SAUDE_CADASTRO` proposed) gating CRUD.
3. **Código range** — confirm the transaction's exact `LocCod` bound (assumed 0..999999 from N(6,0)).
4. **Live-DB confirmation** — verify PG types/nullability match the reorg DDL (`integer` PK + FK,
   `VARCHAR(50)` nome); introspection not run here.
5. **SYS_MUN availability** — it is an un-migrated SYS_* table; the FK is a raw-id column until SYS_MUN
   is migrated. Seed a minimal `SYS_MUN(MunCod, MunNom, MunUF, MunIBGE)` stub in the baseline for the IT.
