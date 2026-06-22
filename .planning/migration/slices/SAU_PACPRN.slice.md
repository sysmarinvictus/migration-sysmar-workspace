# SLICE-SPEC — SAU_PACPRN (Prontuário do Paciente)

> **DECISION: COLLAPSED INTO SAU_PAC (Wave 6) — 2026-06-17**
>
> SAU_PACPRN has no physical DB table; it edits `PacProNum VARCHAR(10)` on `SAU_PAC`.
> The team decided (OQ3, Option A) to include `numeroProntuario` as one field on the
> `PacienteDTO` / `Paciente` entity when the SAU_PAC slice is migrated. This spec is
> retained as an audit trail of the extraction and the mined rules (R1–R17); the rules
> apply to the prontuário field handling within the SAU_PAC service.
>
> **Do NOT run `/migrate-slice SAU_PACPRN`.** The mined rules below should be folded into
> the SAU_PAC SLICE-SPEC at extraction time.
>
> **V1__baseline.sql:** `PacPesNom VARCHAR(50)` and `PacProNum VARCHAR(10)` were added to
> the `SAU_PAC` stub as a result of this extraction — those amendments remain valid.

---

> ~~CRITICAL STRUCTURAL FINDING (original):~~ SAU_PACPRN is a GeneXus Transaction object
> whose base table is `SAU_PAC`. Every cursor in `sau_pacprn__default` (impl:1847–1860)
> operates on `SAU_PAC`. No physical `SAU_PACPRN` table exists.

```yaml
slice: SAU_PACPRN
domain: prontuario
description: >
  Prontuário — manages the patient medical-record number (PacProNum) on SAU_PAC.
  Accepts PacPesCod as a URL parameter, displays the patient name read-only from SYS_PES,
  and lets the operator insert, change, or clear the prontuário number. No standalone table;
  base table is SAU_PAC. After any successful write the legacy app redirects to hwwsau_pac.
wave: 1
complexity: S           # sau_pacprn_impl.java ~78 KB / 2 074 lines

# -----------------------------------------------------------------------
# NOTE: primary_table is SAU_PAC — there is no physical SAU_PACPRN table.
# The GX transaction name is a UI/business-logic label, not a schema object.
# Verified by: GXSPC002/GEN12/NVG/SAU_PACPRN.xml → <BaseTable>SAU_PAC</BaseTable>
#              GXSPC002/GEN12/SAU_PACPRN.sp0 → table_id 3 = SAU_PAC
#              sau_pacprn_impl.java → all cursors (T00102–T001015) target SAU_PAC
# -----------------------------------------------------------------------
primary_table: SAU_PAC

constellation:
  transaction: sau_pacprn.java            # 1 139 bytes; WebServlet stub
  impl: sau_pacprn_impl.java              # 78 KB / 2 074 lines — rule source
  bc: null                                # no sau_pacprn_bc.java
  workwith: null                          # no hwwsau_pacprn.java (invoked from hwwsau_pac)
  view: null                              # no hviewsau_pacprn.java
  prompts: []                             # no hprompt*.java
  sections: []
  sdt: null                               # no SdtSAU_PACPRN.java

# -----------------------------------------------------------------------
# DEPENDS_ON
# Source: Metadata/TableAccess/SAU_PACPRN.xml
# SAU_RECESP listed because delete is blocked when a RECESP row references
# the same PacPesCod (R7, impl line 1210-1218, cursor T001014).
# Cross-cutting (SAU_LOG, SAU_USU, SAU_PRF) not listed here — handled by common/.
# -----------------------------------------------------------------------
depends_on:
  - SAU_PAC      # primary write target — PK (PacPesCod) + PacProNum column
  - SYS_PES      # JOIN for PacPesNom display (not yet migrated; Wave-0 foundation)
  - SAU_RECESP   # reverse-ref delete guard (Wave-6; Portaria 344/98)

# -----------------------------------------------------------------------
# FIELDS
# Source: gxmetadata/sau_pac.json (attributes 3=PacPesCod, 695=PacProNum, and PacPesNom)
#         + sau_pacprn_impl.java variable declarations + SQL cursors
# Only the three attributes that this transaction touches are listed here.
# The full SAU_PAC field set belongs to the SAU_PAC slice (Wave 6).
# -----------------------------------------------------------------------
fields:
  - gx: PacPesCod
    col: PacPesCod         # column on SAU_PAC
    name: id
    type: Long             # GX int, length=14 → Long (baseline already BIGINT)
    nullable: false
    pk: true
    note: "PK; also FK to SYS_PES.PesCod. Disabled/read-only in the GeneXus form Start event (impl:624-630). Passed as URL param AV7PacPesCod."

  - gx: PacProNum
    col: PacProNum         # column on SAU_PAC
    name: numeroProntuario
    type: String
    len: 10                # GX char(10); cursor setString(1, …, 10) at impl:2011,2029
    nullable: true         # initialized '' in initializeNonKey103 (impl:1468)
    pk: false
    note: "The only user-editable field. No format constraint in legacy code (R16, medium confidence). Optimistic-concurrency shadow Z695PacProNum (R6, impl:1015-1027)."

  - gx: PacPesNom
    col: PacPesNom         # column on SAU_PAC (denormalised copy from SYS_PES)
    name: nomePaciente
    type: String
    len: 50                # GX svchar(50)
    nullable: true
    pk: false
    readOnly: true
    source: SYS_PES        # fetched via JOIN on PesCod=PacPesCod (cursors T00104/T00105)
    note: "Display-only. Not written by this transaction. Map insertable=false, updatable=false in the projection entity."

keys:
  pk: [PacPesCod]
  unique: []               # no evidence of unique(PacProNum) constraint; see OQ1
  fks:
    - cols: [PacPesCod]
      ref: SYS_PES         # enforced via checkExtendedTable103 (impl:724-735)

# -----------------------------------------------------------------------
# ENDPOINTS
# No hwwsau_pacprn list or hviewsau_pacprn detail file exists.
# This transaction is invoked from hwwsau_pac (After-Trn redirect, impl:640-641).
# REST surface: 1-to-1 with patient (one PacProNum slot per PacPesCod).
# -----------------------------------------------------------------------
endpoints:
  - { method: GET,    path: /api/prontuarios/{pacienteId}, from: "SAU_PACPRN GET mode (impl:789)", role: SAUDE_CADASTRO, desc: "Fetch prontuário number for a patient" }
  - { method: POST,   path: /api/prontuarios,              from: "SAU_PACPRN INS (impl:1030)",      role: SAUDE_CADASTRO, desc: "Assign PacProNum to patient without one" }
  - { method: PUT,    path: /api/prontuarios/{pacienteId}, from: "SAU_PACPRN UPD (impl:1085)",      role: SAUDE_CADASTRO, desc: "Update PacProNum for existing record" }
  - { method: DELETE, path: /api/prontuarios/{pacienteId}, from: "SAU_PACPRN DLT (impl:1144)",      role: SAUDE_CADASTRO, desc: "Remove prontuário; blocked when SAU_RECESP references patient" }

# -----------------------------------------------------------------------
# PHI FIELDS
# PacPesNom = patient full name (LGPD Art. 11 sensitive health data)
# numeroProntuario = medical record number (health identifier; treat as PHI)
# id = patient surrogate key; pseudonymous alone but PHI in any join context
# -----------------------------------------------------------------------
phi_fields: [id, numeroProntuario, nomePaciente]

auth:
  roles_required: [SAUDE_CADASTRO]     # INFERRED — GeneXus IntegratedSecurity is disabled (impl:1589-1601). See OQ2.
  notes: >
    Legacy auth is custom session-cookie (GX_SESSION_ID) + HMAC-signed 'hsh' field (impl:419-427).
    No acessamodulo call within this transaction — access control delegated to hwwsau_pac caller.
    Role SAUDE_CADASTRO is an inference; verify against acessamodulo.java or the GeneXus KB
    security matrix before implementing @PreAuthorize.

parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  legacy_servlet: "/servlet/sau_pacprn"
  scenarios:
    - get by pacienteId (record exists — PacProNum populated)
    - get by pacienteId (patient exists but PacProNum empty — INS mode)
    - get by pacienteId (patient not found — 404)
    - insert valid (PacProNum assigned to patient without one)
    - insert duplicate (patient already has PacProNum — 409 Conflict)
    - update PacProNum (UPD mode — 200 OK)
    - delete unused (patient not referenced by SAU_RECESP — 204)
    - delete referenced (patient has SAU_RECESP rows — 409 Conflict, "Receituário Controle Especial")
    - concurrent update conflict (Z695PacProNum mismatch — 409)

status: collapsed      # not a standalone slice — PacProNum folded into SAU_PAC (Wave 6)
```

## Mined rules (gx-rule-miner — all sources: sau_pacprn_impl.java)

- **R1** — *No legacy GeneXus role check.* `IntegratedSecurityEnabled()=false`, `IntegratedSecurityLevel()=0` (impl:1589-1601, sau_pacprn.java:33-36). The new service must apply its own JWT/role guard. `kind: authorization` · `confidence: high` · `target: ProntuarioControllerIT#requiresAuthentication`

- **R2** — *Session/JWT required.* Legacy: GX_SESSION_ID cookie + HMAC-signed 'hsh' field validated at impl:96-100, 419-427. REST mapping: mandatory valid JWT Bearer token. `kind: authorization` · `confidence: high` · `target: ProntuarioControllerIT#rejectsUnauthenticatedRequest`

- **R3** — *PacPesCod range.* On POST, submitted patient code must be a valid 14-digit integer ≥ 0 (impl:382-392, GXM_badnum check). `kind: validation` · `confidence: high` · `target: ProntuarioServiceTest#rejectsPatientCodeOutsideValidRange`

- **R4** — *Patient must exist in SYS_PES.* If the join (cursor T00104/T00106/T001013) returns no row, the transaction is blocked with "Não existe 'Paciente'." error. `kind: referential` · `phi: true` · `confidence: high` · `target: ProntuarioServiceTest#rejectsNonExistentPatient`

- **R5** — *One prontuário per patient (INSERT).* If a SAU_PAC row for PacPesCod already exists, the INSERT cursor raises a duplicate-PK error (`GXM_noupdate`, impl:1052-1056, cursor T001010). `kind: validation` · `confidence: high` · `target: ProntuarioServiceTest#rejectsDuplicatePrnForSamePatient`

- **R6** — *Optimistic concurrency.* Before UPDATE or DELETE: re-reads SAU_PAC with FOR UPDATE NOWAIT (cursor T00102). Row locked → `GXM_lock("SAU_PAC")`. Shadow `Z695PacProNum` differs from current DB value → `GXM_waschg("SAU_PAC")` (impl:1003-1028). In REST: return 409 on conflict. `kind: validation` · `confidence: high` · `target: ProntuarioServiceTest#rejectsConcurrentModificationConflict`

- **R7** — *Delete blocked by SAU_RECESP.* Cursor T001014 checks `SELECT 1 FROM SAU_RECESP WHERE PacPesCod = ?`; if a row exists deletion is blocked with `GXM_del("Receituário Controle Especial")` (impl:1210-1218). `kind: referential` · `phi: true` · `confidence: high` · `target: ProntuarioServiceTest#rejectsDeletionWhenPrescriptionExists`

- **R8** — *INSERT defaults.* On INSERT the cursor populates all non-key SAU_PAC columns with fixed defaults (PacUniCod=0, PacSit=0, PacUsuLogin='', PacObi=0, etc. — impl:1855, cursor T001010). Only PacProNum and PacPesCod come from user input. **HIGH RISK for OQ3** — this path creates a near-empty patient shell record. `kind: default` · `confidence: high` · `target: ProntuarioServiceTest#insertsWithCorrectDefaultValues`

- **R9** — *UPDATE writes only PacProNum.* `UPDATE SAU_PAC SET PacProNum=? WHERE PacPesCod=?` (cursor T001011, impl:1856). No other SAU_PAC columns are modified. `kind: formula` · `phi: true` · `confidence: high` · `target: ProntuarioServiceTest#updateWritesOnlyPacProNum`

- **R10** — *PacPesNom is read-only derived.* Patient name is fetched from SYS_PES (cursors T00104/T00105) and never written by this transaction. `kind: derivation` · `phi: true` · `confidence: high` · `target: ProntuarioServiceTest#loadsPatientNameFromSysPes`

- **R11** — *PacPesCod and PacPesNom are display-only in all modes.* Disabled in Start event (impl:624-630); PacPesCod is URL-supplied only. `kind: authorization` · `phi: true` · `confidence: high` · `target: ProntuarioServiceTest#patientCodeIsReadOnlyInAllModes`

- **R12** — *Record-not-found on load.* If no SAU_PAC row for PacPesCod exists in UPD/DLT mode, `GXM_noinsert` is raised (impl:474-480). REST: 404. `kind: validation` · `confidence: high` · `target: ProntuarioServiceTest#rejectsLoadOfNonExistentProntuario`

- **R13** — *PK immutability.* If PacPesCod is altered after load, the key is reset and the operation is blocked with `GXM_getbeforeupd` (impl:906-914). In REST this cannot happen (PK is the path param), but the service must not allow updating a different PacPesCod than the path. `kind: validation` · `confidence: high` · `target: ProntuarioServiceTest#rejectsPrimaryKeyChangeOnUpdate`

- **R14** — *After-Trn redirect to patient list.* After successful write, GeneXus redirects to hwwsau_pac (impl:540-556, 633-643). REST mapping: return updated resource in body; frontend navigates. `kind: side-effect` · `confidence: high` · `target: (no test needed — frontend navigation)`

- **R15** — *Audit MISSING in legacy; MUST be added in new service.* No SAU_LOG write found anywhere in sau_pacprn_impl.java (confirmed by full read). Because PacProNum is a PHI-adjacent identifier, the new service MUST add audit entries for every insert/update/delete via `common/audit` (LGPD Art. 11). `kind: audit` · `phi: true` · `confidence: high` · `target: ProntuarioServiceTest#emitsAuditEventOnEveryWrite`

- **R16** — *PacProNum format: no constraint in legacy.* Field rendered as maxlength=10, no mask (impl:332), no pattern validation in code. Whether it must be numeric-only or follow a convention is unknown. `kind: format` · `confidence: medium` · `target: ProntuarioServiceTest#acceptsAlphanumericProntuarioNumber` · **verify in KB (OQ4)**

- **R17** — *CSRF/session integrity.* Hidden 'hsh' field is HMAC-verified on POST (impl:419-427). In stateless JWT REST, CSRF is not needed for pure API callers; no equivalent check needed. `kind: authorization` · `confidence: high` · `target: (no test needed — JWT replaces session/HMAC)`

## Schema notes (gx-schema-mapper)

- **No SAU_PACPRN table exists.** Entity maps to `@Table(name = "SAU_PAC")`.
- Use a **narrow JPA projection entity** (`ProntuarioPaciente`) with only the three touched columns. When SAU_PAC slice is migrated (Wave 6), merge into the full `Paciente` entity.
- `PacPesNom` must be `insertable=false, updatable=false` — display-only copy.
- **V1__baseline.sql amendment required:** The `SAU_PAC` stub (line 337) currently has only 4 columns; it is missing `PacProNum VARCHAR(10)` and `PacPesNom VARCHAR(50)`. Add both to the stub so Testcontainers integration tests pass `hibernate.ddl-auto=validate`.

```sql
-- V1__baseline.sql — amend SAU_PAC stub:
CREATE TABLE IF NOT EXISTS SAU_PAC (
    PacPesCod           BIGINT  NOT NULL,
    PacUniCod           INTEGER,
    PacPesCadAltUniCod  INTEGER,
    PacPesCadInsUniCod  INTEGER,
    PacPesNom           VARCHAR(50),   -- display copy from SYS_PES; needed by ProntuarioPaciente
    PacProNum           VARCHAR(10),   -- managed by this slice
    CONSTRAINT pk_sau_pac PRIMARY KEY (PacPesCod)
);
```

All column types are **unverified against the live DB** (no psql introspection performed). `hibernate.ddl-auto=validate` will surface mismatches at startup.

## Open questions (resolve before `/migrate-slice SAU_PACPRN`)

1. **OQ1 — PacProNum uniqueness (design).** R5 only blocks duplicate PacPesCod (PK). Is PacProNum itself meant to be unique across all patients (one number per person across the municipality)? The INSERT cursor adds no `UNIQUE` constraint. If it should be unique, add a UK and a pre-insert check to the service. Verify in GeneXus KB or with the domain lead.

2. **OQ2 — Exact permission name.** GeneXus IS is disabled for this transaction. `SAUDE_CADASTRO` is inferred. Confirm via `acessamodulo.java` or the GeneXus security matrix before implementing `@PreAuthorize`.

3. **OQ3 — BLOCKING design decision: standalone endpoint vs. SAU_PAC field.**
   - The INSERT path (R8) creates a near-empty SAU_PAC row with all patient demographics zeroed. This bypasses the full patient validation in the forthcoming SAU_PAC slice — a data quality risk.
   - **Option A (recommended):** Collapse `PacProNum` into the SAU_PAC slice (Wave 6). Do NOT migrate this as a standalone slice; treat `numeroProntuario` as one field on the Paciente DTO. Eliminates the shell-record risk entirely.
   - **Option B:** Keep as standalone `/api/prontuarios` endpoints but REQUIRE that the SAU_PAC row (i.e. the full patient record) already exists before assigning a prontuário number. This means the INSERT endpoint becomes a `PUT` or sub-resource `PATCH` rather than a full row insert. The Testcontainers baseline must seed a full SAU_PAC row to make tests work.
   - **The team must decide before code is generated.** If Option A is chosen, this SLICE-SPEC is obsolete.

4. **OQ4 — PacProNum format.** No pattern enforced in legacy (R16, medium confidence). Check with domain lead: numeric-only? year-prefix? sequential?

5. **OQ5 — LGPD audit obligation.** Legacy emits NO audit log for prontuário changes (R15). Confirm with the LGPD compliance lead whether every PacProNum insert/update/delete must be audit-logged, and which fields (actor, action, entity, old value, new value).

6. **OQ6 — SYS_PES migration state.** PacPesNom is fetched from SYS_PES (JOIN, not yet migrated — Wave-0 foundation). Testcontainers integration tests must seed SYS_PES for the name-derivation tests to pass. Confirm that the SYS_PES stub in V1__baseline.sql is sufficient.
