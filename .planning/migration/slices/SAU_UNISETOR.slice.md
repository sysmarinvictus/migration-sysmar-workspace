# SLICE-SPEC — SAU_UNISETOR (Setor da Unidade)

> Sector/department catalog scoped to a health unit. Each (UniCod, SetorCod) pair identifies
> a sector within a given Unidade de Atendimento. No PHI.
> The V1 baseline has a MINIMAL stub for SAU_UNISETOR (2 columns + PK + FK to SAU_UNI).
> A forward Flyway migration must add the remaining 6 columns before Hibernate validate can pass.

```yaml
slice: SAU_UNISETOR
domain: setor
description: Setor/Departamento da Unidade de Atendimento — sector / department within a health unit
wave: 2
complexity: M           # sau_unisetor_impl.java ~156 KB / 3 087 lines
primary_table: SAU_UNISETOR

constellation:
  transaction:   sau_unisetor.java                  # 49 lines; WebServlet stub
  impl:          sau_unisetor_impl.java              # 155 726 B / 3 087 lines — rule source
  bc:            null                               # not a GeneXus Business Component
  workwith:      null                               # no hwwsau_unisetor; list embedded in SAU_UNI UI
  view:          null                               # no hviewsau_unisetor
  prompts:
    - hpromptsau_unisetor.java                      # 1 196 B stub — "Selecionar Setor" (lookup)
    - hpromptsau_unisetor_impl.java                 # 87 109 B — grid filtered by UniCod + SetorNom LIKE
  web_component:
    - hsau_remsau_unisetorwc.java                   # 1 223 B stub
    - hsau_remsau_unisetorwc_impl.java              # 96 810 B — manages SAU_REM_UNISETOR in the UI
  sdt:           null

# SAU_UNI (parent) must be migrated before this slice.
# The four delete-guard tables need only their minimal stubs to exist for the FK checks to work.
depends_on:
  - SAU_UNI       # UniCod FK — parent; status: planned (Wave 2) — must precede this slice
  - SAU_PAR5      # delete guard only (ParSalUniCod + ParSalSetorCod → UniCod + SetorCod)
  - SAU_USUUNI1   # delete guard only (UniUsuCod + UsuSetorCod → UniCod + SetorCod)
  - SAU_REMLOT    # delete guard only (RemUniCod + RemSetorCod → UniCod + SetorCod)
  - SAU_REM_UNISETOR  # delete guard only (RemUniSetorUniCod + RemUniSetorSetorCod → UniCod + SetorCod)

# ─────────────────────────────────────────────────────────────────────────────
# FIELDS
# Sources: SAU_UNISETORConversion.xml (authoritative DDL) + SQL cursors in
# sau_unisetor_impl.java lines 2877–2893 + Java field declarations lines 2619–2793.
# All sources in full agreement.
# ─────────────────────────────────────────────────────────────────────────────
fields:
  # ── OWN stored columns (8 total) ──────────────────────────────────────────
  - { gx: UniCod,            col: UniCod,            name: unidadeId,     type: Integer, pk: true,  nullable: false,
      note: "Composite PK part 1; FK → SAU_UNI.UniCod. User-entered — not auto-assigned. Range 0–999999 (R1)." }

  - { gx: SetorCod,          col: SetorCod,           name: setorId,       type: Integer, pk: true,  nullable: false,
      note: "Composite PK part 2. User-entered integer. NOT an FK to SAU_SETOR (no FK constraint in GX DDL — see OQ1). Range 0–999999 (R2)." }

  - { gx: SetorNom,          col: SetorNom,           name: nome,          type: String,  len: 50,   nullable: false,
      note: "Sector name. Stored uppercase — GXutil.upper() at impl:685 (R11). VARCHAR(50) NOT NULL in DB." }

  - { gx: SetorEstocador,    col: SetorEstocador,     name: estocador,     type: Byte,    len: 1,    nullable: false,
      note: "Stock-holder flag. SMALLINT NOT NULL in DB. Range 0–9 (R3). Functionally boolean (0=no, 1=yes) — see OQ2." }

  - { gx: SetorSituacao,     col: SetorSituacao,      name: situacao,      type: String,  len: 40,   nullable: false,
      note: "Situation. VARCHAR(40) NOT NULL in DB. Controlled vocab: 'ativo' / 'inativo'. Empty string '' is a filter sentinel (combo 'TODOS') — not persisted (R19, see OQ3)." }

  - { gx: SetorDataInativo,  col: SetorDataInativo,   name: dataInativo,   type: LocalDateTime, nullable: false,
      note: "Date/time the sector was inactivated. TIMESTAMP NOT NULL in DB. GX nullDate (1900-01-01) means 'not inactivated yet' — map to null in DTO/service (R4, see OQ6)." }

  - { gx: SetorHorIni,       col: SetorHorIni,        name: horarioInicio, type: LocalDateTime, nullable: false,
      note: "Sector opening time. TIMESTAMP NOT NULL in DB. GX nullDate = not set. See OQ4/OQ5." }

  - { gx: SetorHorFim,       col: SetorHorFim,        name: horarioFim,    type: LocalDateTime, nullable: false,
      note: "Sector closing time. TIMESTAMP NOT NULL in DB. GX nullDate = not set. See OQ4/OQ5." }

  # ── DERIVED display fields (NOT stored in SAU_UNISETOR; fetched from SAU_UNI JOIN) ──────────
  - { gx: UniNom,   name: unidadeNome,     type: String, stored: false,
      source: "SAU_UNI.UniNom via cursor T001B4/T001B5/T001B13 (impl:2879-2888)",
      note: "Read-only display. edtUniNom_Enabled=0 (impl:2226). Include in response DTO only." }

  - { gx: UniCnes,  name: unidadeCnes,     type: Integer, stored: false,
      source: "SAU_UNI.UniCnes via cursor T001B4 (impl:2879)",
      note: "Read-only display. edtUniCnes_Enabled=0 (impl:2224). Include in response DTO only." }

  - { gx: UniSit,   name: unidadeSituacao, type: Byte, stored: false,
      source: "SAU_UNI.UniSit via cursor T001B4 (impl:2879)",
      note: "Read-only display. 1=ATIVADO / 2=DESATIVADO. cmbUniSit.setEnabled(0) (impl:2222). Include in response DTO only." }

keys:
  pk: [UniCod, SetorCod]
  unique: []
  # GX DDL index: IUNIDADESETOR (impl:SAU_UNISETORConversion.xml) — already is the PK constraint
  fks:
    - { cols: [UniCod], ref: SAU_UNI, constraint: IUNIDADESETOR1,
        note: "UniCod FK to SAU_UNI — confirmed in Conversion XML and impl:1081" }
    # SetorCod has NO FK to SAU_SETOR — Conversion XML shows only one FK constraint.
    # The V1 baseline stub SAU_SETOR table is NOT referenced by a FK from SAU_UNISETOR (see OQ1).
  referenced_by:
    - { table: SAU_PAR5,         cols: [ParSalUniCod, ParSalSetorCod], note: "DELETE guard R12" }
    - { table: SAU_USUUNI1,      cols: [UniUsuCod, UsuSetorCod],       note: "DELETE guard R13" }
    - { table: SAU_REMLOT,       cols: [RemUniCod, RemSetorCod],       note: "DELETE guard R14" }
    - { table: SAU_REM_UNISETOR, cols: [RemUniSetorUniCod, RemUniSetorSetorCod], note: "DELETE guard R15; this table also needs column extension — see schema notes" }

endpoints:
  # No hwwsau_unisetor or hviewsau_unisetor; list is embedded in the SAU_UNI screen.
  # REST surface nests under /api/unidades/{unidadeId}/setores per NAMING.md.
  - { method: GET,    path: /api/unidades/{unidadeId}/setores,             role: SAUDE_CADASTRO,
      from: "hpromptsau_unisetor_impl.java grid (line 279, cursor H002I2)",
      desc: "List all setores for a given unidade; optional SetorNom LIKE filter; paginated." }
  - { method: GET,    path: /api/unidades/{unidadeId}/setores/{setorId},   role: SAUDE_CADASTRO,
      from: "sau_unisetor_impl.java INS/UPD load (impl:771-783)",
      desc: "Get single setor by composite key; response includes UniNom/UniCnes/UniSit derived fields." }
  - { method: POST,   path: /api/unidades/{unidadeId}/setores,             role: SAUDE_CADASTRO,
      from: "sau_unisetor_impl.java insert1B55() (impl:1575, cursor T001B10 impl:2885)",
      desc: "Create new setor for the unidade; composite PK is fully user-entered." }
  - { method: PUT,    path: /api/unidades/{unidadeId}/setores/{setorId},   role: SAUDE_CADASTRO,
      from: "sau_unisetor_impl.java update1B55() (impl:1630, cursor T001B11 impl:2886)",
      desc: "Update setor mutable fields (SetorNom, SetorEstocador, SetorSituacao, SetorDataInativo, SetorHorIni, SetorHorFim)." }
  - { method: DELETE, path: /api/unidades/{unidadeId}/setores/{setorId},   role: SAUDE_CADASTRO,
      from: "sau_unisetor_impl.java delete() (impl:1685, cursor T001B12 impl:2887)",
      desc: "Delete setor; guarded by 4 referential checks (R12-R15)." }
  - { method: GET,    path: /api/unidades/{unidadeId}/setores/lookup,      role: SAUDE_CADASTRO,
      from: "hpromptsau_unisetor_impl.java Selecionar Setor grid (impl:621)",
      desc: "Autocomplete/lookup list for selecting a setor (SetorCod + SetorNom); optional q= filter." }

phi_fields: []
# SAU_UNISETOR is a facility/operational catalog. No patient identifiers, CPF, CNS,
# diagnoses, or medication data. UniNom/UniCnes are facility identifiers, not personal data.
# No LGPD Art. 5/11 PHI present. @Auditable still required per project mandate (R17).

auth:
  roles_required: [SAUDE_CADASTRO]
  notes: >
    IntegratedSecurityEnabled() returns false in both sau_unisetor.java (line 36) and
    hpromptsau_unisetor.java (line 39). No explicit pisauthorized() call found. Access
    is likely controlled at the parent SAU_UNI screen level. The conventional role
    SAUDE_CADASTRO is used as a placeholder — confirm the exact @PreAuthorize expression
    against the GeneXus KB Permissions tab for SAU_UNISETOR (see OQ5).

parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios:
    - "POST valid (all fields) — verify all 8 columns stored; UniNom/UniCnes/UniSit derived in response"
    - "POST minimal (UniCod + SetorCod + SetorNom + SetorEstocador + SetorSituacao) — accepted"
    - "POST duplicate (UniCod + SetorCod) already exists → 409 Conflict"
    - "POST with non-existent UniCod → 422 FK error"
    - "PUT update SetorNom → SetorNom stored uppercase"
    - "PUT update SetorSituacao='inativo'; verify SetorDataInativo set to non-null"
    - "PUT on non-existent composite key → 404"
    - "PUT — optimistic concurrency: concurrent update of same row → 409"
    - "DELETE unreferenced setor → 204"
    - "DELETE blocked by SAU_PAR5 → 409 (Configuração Unidade Saldo de Medicamento)"
    - "DELETE blocked by SAU_USUUNI1 → 409 (Usuário/Unidade/Setor)"
    - "DELETE blocked by SAU_REMLOT → 409 (Lote de Medicamento)"
    - "DELETE blocked by SAU_REM_UNISETOR → 409"
    - "GET list filtered by SetorNom LIKE — verify pagination"
    - "GET lookup for UniCod=X — verify only setores for that unit returned"
    - "GET /api/unidades/{id}/setores/{id} — 404 for unknown composite key"

status: verified       # parity 2026-06-22: 12/12 live PARITY + 3 guards code-verified + 1 deferred. Report: backend/src/test/resources/parity/setor/PARITY-REPORT.md.
# Entity↔live types all match. PUT inativo sets SetorDataInativo; nome UPPERCASE; delete-guard (REM_UNISETOR) → 409 (seeded).
# DEFERRED: optimistic concurrency (no @Version column — RD1). Delete-guards PAR5/USUUNI1/REMLOT code-verified (queries reviewed; mechanism proven via R15). No business divergences.
```

## Mined rules (gx-rule-miner — 20 rules; all sources: sau_unisetor_impl.java unless noted)

- **R1** — *UniCod range guard.* UniCod must be integer 0–999999; out-of-range value triggers numeric-format error before form processing (impl:645-654). New service: `@Min(0) @Max(999999)`. `kind: validation` · `confidence: high` · `target: UniSetorServiceTest#rejectsUniCodOutOfRange`

- **R2** — *SetorCod range guard.* SetorCod must be integer 0–999999 (impl:670-678). New service: `@Min(0) @Max(999999)`. `kind: validation` · `confidence: high` · `target: UniSetorServiceTest#rejectsSetorCodOutOfRange`

- **R3** — *SetorEstocador range guard.* SetorEstocador must be 0–9 (single GX byte digit, impl:687-695). New service: `@Min(0) @Max(9)`. `kind: validation` · `confidence: high` · `target: UniSetorServiceTest#rejectsSetorEstocadorOutOfRange`

- **R4** — *SetorDataInativo datetime format.* Non-null SetorDataInativo must be a valid datetime; invalid format rejected with bad-datetime error; field reset to null/nullDate (impl:705-713). New service: accept `LocalDateTime` or null; reject unparseable string at DTO binding level. `kind: validation` · `confidence: high` · `target: UniSetorServiceTest#rejectsInvalidSetorDataInativo`

- **R5** — *SetorHorIni datetime format.* Non-null SetorHorIni must be a valid datetime (impl:720-728). Same as R4. `kind: validation` · `confidence: high` · `target: UniSetorServiceTest#rejectsInvalidSetorHorIni`

- **R6** — *SetorHorFim datetime format.* Non-null SetorHorFim must be a valid datetime (impl:735-743). Same as R4. `kind: validation` · `confidence: high` · `target: UniSetorServiceTest#rejectsInvalidSetorHorFim`

- **R7** — *UniCod FK existence check.* Non-zero UniCod must exist in SAU_UNI; if missing, rejected with "Não existe Unidade de Atendimento." (cursors T001B4/T001B6/T001B13, impl:1078-1085, 1112-1116). `kind: referential` · `confidence: high` · `target: UniSetorServiceTest#rejectsUnidadeNotFound`

- **R8** — *UniNom/UniCnes/UniSit are derived read-only.* These fields are fetched from SAU_UNI by UniCod (cursor T001B4, impl:1086-1094); all three UI controls are disabled (`edtUniNom_Enabled=0` impl:2226, `edtUniCnes_Enabled=0` impl:2224, `cmbUniSit.setEnabled(0)` impl:2222). Must appear in response DTO only — never accepted in request body. `kind: derivation` · `confidence: high` · `target: UniSetorServiceTest#unidadeDisplayFieldsAreReadOnly`

- **R9** — *Composite PK unique on INSERT.* Inserting a (UniCod, SetorCod) pair that already exists triggers duplicate-PK error (cursor T001B10 returns status 1, impl:1597-1600). New service: catch `DataIntegrityViolationException` → 409. `kind: validation` · `confidence: high` · `target: UniSetorServiceTest#rejectsDuplicatePrimaryKey`

- **R10** — *Optimistic concurrency on UPDATE/DELETE.* The 6 mutable columns (SetorNom, SetorEstocador, SetorSituacao, SetorDataInativo, SetorHorIni, SetorHorFim) are re-read with `FOR UPDATE NOWAIT` (cursor T001B2, impl:1513-1572). Row locked → `GXM_lock`; any column changed since load → `GXM_waschg` → 409. New service: implement with `@Version` (requires adding a version column via Flyway — deferred, see OQ8). `kind: validation` · `confidence: high` · `target: UniSetorServiceTest#rejectsStaleUpdateOnConcurrentChange` — `@Disabled` stub`

- **R11** — *SetorNom stored uppercase.* `A100SetorNom = GXutil.upper(httpContext.cgiGet(...))` (impl:685); also `onblur: this.value=this.value.toUpperCase()` in form (impl:405). Service must apply `.toUpperCase().trim()` before persist. `kind: derivation` · `confidence: high` · `target: UniSetorServiceTest#setorNomIsUppercasedOnSave`

- **R12** — *DELETE blocked by SAU_PAR5.* If any SAU_PAR5 row has `(ParSalUniCod, ParSalSetorCod) = (UniCod, SetorCod)`, delete is rejected: "Configuração Unidade Saldo de Medicamento" (cursor T001B14, impl:1769-1774). `kind: referential` · `confidence: high` · `target: UniSetorServiceTest#rejectsDeleteWhenReferencedBySauPar5`

- **R13** — *DELETE blocked by SAU_USUUNI1.* If any SAU_USUUNI1 row has `(UniUsuCod, UsuSetorCod) = (UniCod, SetorCod)`, delete is rejected: "Usuário/Unidade/Setor" (cursor T001B15, impl:1775-1781). `kind: referential` · `confidence: high` · `target: UniSetorServiceTest#rejectsDeleteWhenReferencedBySauUsuUni1`

- **R14** — *DELETE blocked by SAU_REMLOT.* If any SAU_REMLOT row has `(RemUniCod, RemSetorCod) = (UniCod, SetorCod)`, delete is rejected: "Lote de Medicamento" (cursor T001B16, impl:1783-1789). `kind: referential` · `confidence: high` · `target: UniSetorServiceTest#rejectsDeleteWhenReferencedBySauRemLot`

- **R15** — *DELETE blocked by SAU_REM_UNISETOR.* If any SAU_REM_UNISETOR row has `(RemUniSetorUniCod, RemUniSetorSetorCod) = (UniCod, SetorCod)`, delete is rejected: "SAU_REM_UNISETOR" (cursor T001B17, impl:1791-1797). `kind: referential` · `confidence: high` · `target: UniSetorServiceTest#rejectsDeleteWhenReferencedBySauRemUnisetor`

- **R16** — *Authorization — no built-in GX security.* `IntegratedSecurityEnabled()` returns false (sau_unisetor_impl.java:2401-2403, sau_unisetor.java:33-36). No `pisauthorized()` call found. New service must enforce Spring Security JWT + `@PreAuthorize`. See OQ5. `kind: authorization` · `confidence: medium` · `target: UniSetorControllerIT#rejectsUnauthenticatedAccess`

- **R17** — *No legacy audit trail.* No `psau_inc_log` / SAU_LOG call anywhere in sau_unisetor_impl.java. New service must add `@Auditable` on all write operations per the LGPD project mandate. `kind: audit` · `confidence: high` · `target: UniSetorServiceTest#auditEventIsPublishedOnWrite`

- **R18** — *Composite PK immutable on UPDATE.* UPDATE cursor T001B11 targets only the 6 mutable columns; any attempt to change UniCod or SetorCod triggers `GXM_getbeforeupd` (impl:1290-1299). In REST: path params `unidadeId` + `setorId` are authoritative; request body must not include PK fields. `kind: validation` · `confidence: high` · `target: UniSetorServiceTest#rejectsPrimaryKeyChangeOnUpdate`

- **R19** — *SetorSituacao controlled vocabulary.* Combo items: `''` (TODOS — filter only), `'ativo'`, `'inativo'` (impl:101-104). Only `'ativo'` and `'inativo'` should be stored; empty string is a filter sentinel. New service: validate via `@NotNull` + `@Pattern` or enum. `kind: validation` · `confidence: medium` · `target: UniSetorServiceTest#rejectsInvalidSetorSituacao`

- **R20** — *UniSit read-only derived.* UniSit (1=ATIVADO / 2=DESATIVADO) fetched from SAU_UNI (impl:92-98, cmbUniSit items); `cmbUniSit.setEnabled(0)` (impl:2222). Include in response DTO only. `kind: derivation` · `confidence: high` · `target: UniSetorServiceTest#unidadeSituacaoIsReadOnly`

## Schema notes (gx-schema-mapper)

### SAU_UNISETOR — authoritative DDL (SAU_UNISETORConversion.xml)
```sql
CREATE TABLE SAU_UNISETOR (
    UniCod             INTEGER      NOT NULL,
    SetorCod           INTEGER      NOT NULL,
    SetorNom           VARCHAR(50)  NOT NULL,
    SetorEstocador     SMALLINT     NOT NULL,
    SetorSituacao      VARCHAR(40)  NOT NULL,
    SetorDataInativo   TIMESTAMP    NOT NULL,
    SetorHorIni        TIMESTAMP    NOT NULL,
    SetorHorFim        TIMESTAMP    NOT NULL,
    CONSTRAINT pk_sau_unisetor PRIMARY KEY (UniCod, SetorCod)
);
ALTER TABLE SAU_UNISETOR ADD CONSTRAINT IUNIDADESETOR1 FOREIGN KEY (UniCod) REFERENCES SAU_UNI (UniCod);
```
**Only one FK constraint** — UniCod → SAU_UNI. No FK to SAU_SETOR exists in the GX DDL.

### V1 baseline gap — MINIMAL stub has only 2 columns
The current stub (V1__baseline.sql lines 264-270) has only `UniCod`, `SetorCod`, PK, and FK to SAU_UNI.
**A Flyway migration (V2__sau_unisetor_extend.sql) must add the 6 missing columns** before Hibernate validate can pass:
```sql
ALTER TABLE SAU_UNISETOR
    ADD COLUMN IF NOT EXISTS SetorNom         VARCHAR(50)  NOT NULL DEFAULT '',
    ADD COLUMN IF NOT EXISTS SetorEstocador   SMALLINT     NOT NULL DEFAULT 0,
    ADD COLUMN IF NOT EXISTS SetorSituacao    VARCHAR(40)  NOT NULL DEFAULT '',
    ADD COLUMN IF NOT EXISTS SetorDataInativo TIMESTAMP    NOT NULL DEFAULT '1900-01-01 00:00:00',
    ADD COLUMN IF NOT EXISTS SetorHorIni      TIMESTAMP    NOT NULL DEFAULT '1900-01-01 00:00:00',
    ADD COLUMN IF NOT EXISTS SetorHorFim      TIMESTAMP    NOT NULL DEFAULT '1900-01-01 00:00:00';
```

### SAU_REM_UNISETOR — also needs extension
Conversion XML (SAU_REM_UNISETORConversion.xml) shows the full table has 6 columns:
`(RemCod, RemUniSetorSeq, RemUniSetorUniCod) PK` + `RemUniSetorSetorCod`, `RemUniSetorEstMin`, `RemUniSetorSit`
and FK: `(RemUniSetorUniCod, RemUniSetorSetorCod) → SAU_UNISETOR(UniCod, SetorCod)`.

The V1 stub (lines 328-334) has only 3 columns. The same V2 migration should also extend it:
```sql
ALTER TABLE SAU_REM_UNISETOR
    ADD COLUMN IF NOT EXISTS RemUniSetorSetorCod  INTEGER,
    ADD COLUMN IF NOT EXISTS RemUniSetorEstMin     INTEGER,
    ADD COLUMN IF NOT EXISTS RemUniSetorSit        SMALLINT;
-- Add FK from SAU_REM_UNISETOR to SAU_UNISETOR (after SAU_UNISETOR is extended)
ALTER TABLE SAU_REM_UNISETOR
    ADD CONSTRAINT IF NOT EXISTS fk_sau_rem_unisetor_setor
        FOREIGN KEY (RemUniSetorUniCod, RemUniSetorSetorCod)
        REFERENCES SAU_UNISETOR (UniCod, SetorCod);
```

### SAU_SETOR table — discrepancy flag
The V1 baseline has a `SAU_SETOR` table stub (lines 257-263). However, **no FK from SAU_UNISETOR to SAU_SETOR exists in the GX DDL** (Conversion XML shows only one FK: UniCod → SAU_UNI). SetorCod in SAU_UNISETOR is a locally-assigned integer PK component, not a reference to a separate Setor entity. The SAU_SETOR stub is inert but harmless. Confirm whether SAU_SETOR is used by other transactions before removing it (see OQ1).

### JPA entity design
- `@IdClass(UniSetorId.class)` — IdClass with `int uniCod; int setorCod;` (implements Serializable)
- `@Table(name = "SAU_UNISETOR")` with `@Column` annotations pinning all physical names
- GX "null dates" (1900-01-01 00:00:00) → `LocalDateTime` fields; service layer converts nullDate → null in DTO
- `SetorSituacao` → `String` field; validate enum at DTO layer; store 'ativo'/'inativo' only (not '')
- Derived fields (UniNom, UniCnes, UniSit) → NOT entity fields; fetched via `enrich()` in service using `UnidadeRepository.findById(uniCod)`

## Open questions (resolve before `/migrate-slice SAU_UNISETOR`)

1. **OQ1 — SAU_SETOR FK relationship.** The Conversion XML shows NO FK from SAU_UNISETOR to SAU_SETOR. SetorCod appears to be a locally-scoped integer (e.g., Sector 1, 2, 3 within each Unidade), not a reference to a separate entity. Confirm in GeneXus KB: is SetorCod a free numeric attribute or does it reference a separate SAU_SETOR object? If free, the SAU_SETOR stub in V1 baseline can be dropped from future migrations.

2. **OQ2 — SetorEstocador semantics.** The GX picture is '9' (0–9 range, R3). Is value 0=não/1=sim effectively boolean, or do values 2–9 have distinct meanings (e.g., different stockkeeper roles)? Verify with functional spec or KB before modelling as `boolean` vs `byte`.

3. **OQ3 — SetorSituacao empty string.** Combo defines '', 'ativo', 'inativo'. Is the empty string ever actually stored in the DB (e.g., for new unsaved records), or is it strictly a filter sentinel? If never stored, the DB column could use a CHECK constraint — but confirm before adding one.

4. **OQ4 — SetorHorIni/SetorHorFim type.** Both are GX datetime (TIMESTAMP in DB), but they represent operating hours start/end. Is the date component meaningful (they vary day-to-day), or is only the time-of-day portion used? If time-only, `LocalTime` would be cleaner than `LocalDateTime`, but the DB stores TIMESTAMP — map to `LocalDateTime` and ignore date unless business says otherwise.

5. **OQ5 — Required authorization role.** `IntegratedSecurityEnabled=false`; no `pisauthorized()` call found. Parent screens may gate access via module codes. Query `SAU_PRFCON` or the GeneXus KB Permissions tab for SAU_UNISETOR to find the exact role/permission string. `SAUDE_CADASTRO` is a placeholder.

6. **OQ6 — GX nullDate mapping.** GeneXus stores "empty" datetimes as `1900-01-01 00:00:00` (not SQL NULL) because the columns are NOT NULL. The service layer should: on read, treat nullDate as `null` in the response DTO; on write, accept `null` from the request and store nullDate in the DB. Confirm this is the live DB behavior before implementing.

7. **OQ7 — Additional delete guards from live DB.** Confirm no other tables FK into SAU_UNISETOR beyond the 4 found in the impl. Run against live DB: `SELECT tc.table_name, kcu.column_name FROM information_schema.table_constraints tc JOIN information_schema.key_column_usage kcu USING (constraint_name) JOIN information_schema.referential_constraints rc USING (constraint_name) JOIN information_schema.constraint_column_usage ccu USING (constraint_catalog, constraint_schema, constraint_name) WHERE ccu.table_name = 'SAU_UNISETOR';`

8. **OQ8 — R10 optimistic concurrency deferred.** SAU_UNISETOR has no version/timestamp column; `@Version` (JPA optimistic locking) requires adding a column via Flyway and coordinating with the live DB. Deferred until Wave 2 coordinator approves. Add `@Disabled` stub test for R10 similar to the pattern in DistritoServiceTest.

9. **OQ9 — hsau_remsau_unisetorwc scope.** `hsau_remsau_unisetorwc_impl.java` (96 KB) is a GX web component embedded inside the SAU_UNISETOR UI to manage `SAU_REM_UNISETOR` (which medications/remessas are assigned per sector). Should this web component generate its own nested endpoints (e.g., `/api/unidades/{id}/setores/{id}/remessas`) in this slice, or is it deferred to the SAU_REM_UNISETOR slice? Recommended: defer to avoid scope creep (mark as out-of-scope in this SLICE-SPEC).
