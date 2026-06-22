```yaml
slice: SAU_APRREM
domain: forma-apresentacao             # OQ-1 RESOLVED: renamed to match servlet label + SAU_REM usage; NAMING.md corrected
description: Forma de Apresentação      # sau_aprrem.java:30 getServletInfo()
wave: 3
complexity: S                          # sau_aprrem_impl.java ~91 KB; single-table leaf
primary_table: SAU_APRREM
sub_tables: []

constellation:
  transaction:  sau_aprrem.java                 # servlet stub; getServletInfo()="Forma de Apresentação"
  impl:         sau_aprrem_impl.java            # transaction impl — rule source
  bc:           null                            # no BC; no gxmetadata/sau_aprrem.json
  prompts:
    - hpromptsau_aprrem.java                     # "Selecionar Forma de Apresentação" lookup
  pk_procedure: psau_inc_aprrem.java            # "Gera Código da Apresentação" — next AprRemCod = MAX+1
  metadata:     Metadata/TableAccess/SAU_APRREM.xml
  # No hww*/hview*/section/SDT — pure single-table CRUD via the transaction.

# ── Primary table SAU_APRREM (3 columns — confirmed against the live saude-mandaguari DB)
fields:
  - { gx: AprRemCod, name: id,         col: AprRemCod, type: integer,     pk: true,  nullable: false, note: "PK; legacy generates MAX+1 (psau_inc_aprrem). Use DB SEQUENCE seq_sau_aprrem_cod — OQ-4." }
  - { gx: AprRemDes, name: descricao,  col: AprRemDes, type: varchar(30), pk: false, nullable: true,  note: "Required at service level (R2); stored UPPERCASE (R4). picture @!" }
  - { gx: AprRemAbr, name: abreviacao, col: AprRemAbr, type: char(5),     pk: false, nullable: true,  note: "Required at service level (R3); stored UPPERCASE (R5). CHAR(5) → @JdbcTypeCode(Types.CHAR)." }

keys:
  pk:
    SAU_APRREM: [AprRemCod]
  indexes:
    SAU_APRREM:
      - { name: usau_aprrem_desc, cols: [AprRemCod DESC], unique: false, note: "GeneXus secondary index (present live)" }
  unique: []                           # none in regen or live DB — descricao/abreviacao NOT unique (OQ-2)
  fks: []                              # leaf — owns no outbound FK
  referenced_by:
    - { table: SAU_REM, col: AprRemCod, note: "Medicamento → SAU_APRREM (parent). SAU_REM already migrated; kept as raw-id column apresentacaoCodigo (no @ManyToOne)." }
  delete_guards:
    - "SAU_REM references AprRemCod (R7) — block delete; 409 Conflict. Cursor: SELECT RemCod FROM SAU_REM WHERE AprRemCod = ? LIMIT 1."

endpoints:
  - { method: GET,    path: /api/formas-apresentacao,        from: "sau_aprrem list (cursor T001912)", desc: "list + search by descricao (LIKE)" }
  - { method: GET,    path: /api/formas-apresentacao/{id},   from: "sau_aprrem display (T00193)" }
  - { method: POST,   path: /api/formas-apresentacao,        from: "sau_aprrem INS (T00198)" }
  - { method: PUT,    path: /api/formas-apresentacao/{id},   from: "sau_aprrem UPD (T00199)" }
  - { method: DELETE, path: /api/formas-apresentacao/{id},   from: "sau_aprrem DLT (T001910)" }
  - { method: GET,    path: /api/formas-apresentacao/lookup, from: hpromptsau_aprrem, desc: "autocomplete (AprRemCod + AprRemDes)" }

depends_on: []                         # leaf lookup; owns no outbound FK
# Referenced BY SAU_REM.AprRemCod (SAU_REM migrated + verified). No upstream slices required.

phi_fields: []                         # medication-presentation reference data — no patient/health PHI

auth:
  roles_required: [SAUDE_CADASTRO]     # INFERRED — confirm (OQ-3)
  notes: >
    Legacy enforces pisauthorized("SAU_APRREM", Gx_mode) on Start (sau_aprrem_impl.java:676-707);
    no logged-in user → redirect hnotauthorizeduser; not authorized → hnotauthorizedprompt.
    IntegratedSecurityEnabled()=false, so this app-level check is the sole gate. Map to JWT + per-method
    @PreAuthorize at a cadastro-level write role. Reads available to SAUDE_CADASTRO.

rules:
  - id: R1
    when: insert
    desc: "AprRemCod (PK) auto-generated server-side: MAX(AprRemCod)+1 (1 if empty), in the After-Confirm phase, only when Gx_mode=INS. Use seq_sau_aprrem_cod in the new app (MAX+1 is race-prone)."
    source: "sau_aprrem_impl.java:1393-1402; psau_inc_aprrem.java:54-69,115"
    confidence: high
    test: FormaApresentacaoServiceTest#insertGeneratesNextSequentialCode

  - id: R2
    when: insert|update
    desc: "AprRemDes (descrição) required; blank → 'Informe a Descrição da Forma de Apresentação!'"
    source: "sau_aprrem_impl.java:812-818"
    confidence: high
    test: FormaApresentacaoServiceTest#descricaoIsRequired

  - id: R3
    when: insert|update
    desc: "AprRemAbr (abreviação) required; blank → 'Informe a Abreviação!'"
    source: "sau_aprrem_impl.java:819-825"
    confidence: high
    test: FormaApresentacaoServiceTest#abreviacaoIsRequired

  - id: R4
    when: insert|update
    desc: "AprRemDes stored UPPERCASE (GXutil.upper before persistence)."
    source: "sau_aprrem_impl.java:444, 366"
    confidence: high
    test: FormaApresentacaoServiceTest#descricaoStoredUpperCase

  - id: R5
    when: insert|update
    desc: "AprRemAbr stored UPPERCASE (GXutil.upper before persistence)."
    source: "sau_aprrem_impl.java:447, 377"
    confidence: high
    test: FormaApresentacaoServiceTest#abreviacaoStoredUpperCase

  - id: R6
    when: insert|update
    desc: "AprRemCod must be 0..999999 (GeneXus 6-digit numeric domain); out of range → GXM_badnum. Guards manual entry only (PK is auto-generated). Decide whether to keep the 6-digit ceiling — OQ-6."
    source: "sau_aprrem_impl.java:427-437"
    confidence: medium
    test: FormaApresentacaoServiceTest#codigoRejectsOutOfRange

  - id: R7
    when: delete
    desc: "Block DELETE when referenced by a Medicamento (SAU_REM.AprRemCod). Legacy GXM_del parameterized 'Medicamento'. New: 409 Conflict."
    source: "sau_aprrem_impl.java:1272-1287 (cursor T001911)"
    confidence: high
    test: FormaApresentacaoServiceTest#deleteBlockedWhenReferencedByMedicamento

  - id: R8
    when: insert
    desc: "Duplicate-PK guard (cursor status==1) → GXM_noupdate. Corresponds to the PK uniqueness constraint surfacing as a conflict."
    source: "sau_aprrem_impl.java:1124-1131"
    confidence: high
    test: FormaApresentacaoServiceTest#insertRejectsDuplicatePrimaryKey

  - id: R9
    when: update|delete
    desc: "Record must exist; concurrent change → GXM_waschg, deleted → GXM_recdeleted, locked → GXM_lock (SELECT FOR UPDATE NOWAIT). Port as not-found + optimistic-lock behavior."
    source: "sau_aprrem_impl.java:1016-1022, 1072-1103, 1181-1185"
    confidence: medium
    test: FormaApresentacaoServiceTest#updateRejectsWhenRecordChangedOrDeleted

  - id: R10
    when: always
    desc: "Authorization-gated: authenticated user required; pisauthorized('SAU_APRREM', mode)=0 → not authorized. Map to JWT + @PreAuthorize per operation."
    source: "sau_aprrem_impl.java:676-707, 1767-1780"
    confidence: medium
    test: FormaApresentacaoServiceTest#operationRequiresAuthorizedUser

  - id: R11
    when: insert|update|delete
    desc: "After commit, write audit row via psau_inc_log (EmpCod, LogDat, UsuCod, mode, Pgmname='SAU_APRREM', LogKey=AprRemCod). New: common/audit record per write."
    source: "sau_aprrem_impl.java:1306-1339"
    confidence: high
    test: FormaApresentacaoServiceTest#successfulWriteEmitsAuditLog

  - id: R12
    when: insert
    desc: "Successful insert surfaces GXM_sucadded — translate to 201 Created (behavioral note, not an invariant)."
    source: "sau_aprrem_impl.java:1139"
    confidence: medium
    test: FormaApresentacaoServiceTest#insertReturnsCreated

# ── Persistence (gx-schema-mapper; live DB introspected 2026-06-21) ──────────
entity_mapping:
  class: FormaApresentacao
  table: SAU_APRREM
  columns:
    - { field: id,         column: AprRemCod, javaType: Integer, id: true, generated: "seq_sau_aprrem_cod (manual nextval, per V5 seq_sau_rem_cod convention — NOT @GeneratedValue)" }
    - { field: descricao,  column: AprRemDes, javaType: String, length: 30 }
    - { field: abreviacao, column: AprRemAbr, javaType: String, length: 5, jdbcType: "@JdbcTypeCode(Types.CHAR)  # bpchar(5)" }
  relationships: "none. SAU_REM.apresentacaoCodigo stays a raw-id column (no @ManyToOne) — see OQ-5."

flyway_plan:
  table: "Already covered — V5__sau_rem_support.sql stubs SAU_APRREM(AprRemCod, AprRemDes VARCHAR(30), AprRemAbr CHAR(5), PK). Live DB matches exactly. No CREATE needed."
  forward_migration: "V6__sau_aprrem_support.sql (idempotent): CREATE SEQUENCE IF NOT EXISTS seq_sau_aprrem_cod + setval(MAX+1); CREATE INDEX IF NOT EXISTS usau_aprrem_desc ON SAU_APRREM (AprRemCod DESC)."

parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios:
    - "list default (order by descricao)"
    - "list filter by descricao (partial, case-insensitive)"
    - "GET by id — existing"
    - "GET by id — 404 unknown"
    - "POST valid (descricao + abreviacao) → 201, code generated, both stored UPPERCASE"
    - "POST blank descricao → 422 'Informe a Descrição da Forma de Apresentação!'"
    - "POST blank abreviacao → 422 'Informe a Abreviação!'"
    - "PUT update descricao/abreviacao → 200, UPPERCASE"
    - "DELETE unused → 204"
    - "DELETE referenced by SAU_REM → 409"
    - "lookup by descricao partial (hpromptsau_aprrem)"

status: verified

# Backend + frontend + tests generated, reviewed, and parity-verified on 2026-06-21.
parity_result:
  date: 2026-06-21
  method: "DB-state comparison; new app :8090 vs shared snapshot; legacy Tomcat host.docker.internal:8080"
  report: "backend/src/test/resources/parity/forma-apresentacao/PARITY-REPORT.md (+ scenarios.json)"
  summary: "11/11 PARITY — no divergences"
  notes:
    - "OQ-5 confirmed: SAU_APRREM empty (0 rows) in the snapshot; all scenarios self-seeded (ZZAPR marker) and cleaned up — table restored to 0 rows."
    - "Verified Flyway-disabled + ddl-auto=none; seq_sau_aprrem_cod created manually in the snapshot (V6-equivalent)."
    - "R9 optimistic-lock half remains deferred (RD1) — no version column; acceptable for a lookup table."

# Backend + frontend + tests generated and reviewed on 2026-06-21 (/migrate-slice SAU_APRREM).
# Domain: formaapresentacao/ — FormaApresentacao (SAU_APRREM: id, descricao, abreviacao char(5)).
# Tests: FormaApresentacaoServiceTest 10 (one per rule) + FormaApresentacaoControllerIT 9 (incl. 401/403).
# Full `mvn verify` GREEN (unit 173, integ 246, 0 fail). Frontend tsc clean.
code_review:
  reviewer: migration-reviewer
  date: 2026-06-21
  fixed:
    - "Default sort: GET /api/formas-apresentacao now @PageableDefault(size=20, sort=descricao) — matches reference + parity 'list default order by descricao'."
    - "Test coverage: rewrote unit tests one-per-rule with spec names; added R8 (insertRejectsDuplicatePrimaryKey), R9 (updateRejectsWhenRecordChangedOrDeleted, not-found half), R11 (successfulWriteEmitsAuditLog — CREATE+UPDATE+DELETE)."
    - "R10: added IT forbiddenForUserWithoutRole (authenticated, missing SAUDE_CADASTRO → 403)."
  false_positives:
    - "Reviewer flagged the parity stub as missing — it exists at parity/FormaApresentacaoParityIT.java (conventional package, 11 @Disabled scenarios; ran/skipped in verify)."
  deferred:
    - id: RD1
      finding: "R9 optimistic-lock half (GXM_waschg/GXM_lock) not ported — SAU_APRREM has no version column (consistent with prior slices). Only not-found is enforced; concurrent edits last-writer-wins."
      action: "Add @Version/ETag in a later phase if needed; low risk for a lookup table."
  notes:
    - "Parity fixtures (src/test/resources/parity/forma-apresentacao/) are produced by /verify-parity, not migrate-slice — same as SAU_UNI/SAU_REM."
    - "OQ-5: SAU_APRREM may be empty in the snapshot; parity write-scenarios must seed their own rows."

open_questions:
  - id: OQ-1
    topic: naming
    status: RESOLVED 2026-06-21
    resolution: "Renamed to 'Forma de Apresentação' (recommended): domain forma-apresentacao, endpoint /api/formas-apresentacao, class/test FormaApresentacao*. NAMING.md corrected (aprrem → forma-apresentacao)."
  - id: OQ-2
    topic: business-rule
    status: RESOLVED 2026-06-21
    resolution: "Do NOT add uniqueness on descrição/abreviação — legacy does not enforce it; keep legacy behavior (no unique constraint)."
  - id: OQ-3
    topic: auth
    status: RESOLVED 2026-06-21
    resolution: "Confirmed SAUDE_CADASTRO for all operations (reads + writes). Map pisauthorized('SAU_APRREM') → @PreAuthorize(\"hasRole('SAUDE_CADASTRO')\")."
  - id: OQ-4
    topic: schema
    status: RESOLVED 2026-06-21
    resolution: "Live DB has NO sequence for AprRemCod (MAX+1 in psau_inc_aprrem). Create seq_sau_aprrem_cod in V6 (seeded MAX+1); assign via manual nextval in the service, mirroring SAU_REM's seq_sau_rem_cod."
  - id: OQ-5
    topic: data
    status: RESOLVED 2026-06-21
    resolution: >
      Apresentação data EXISTS in production but is missing from this non-prod snapshot (a table
      not yet carried into the snapshot — expected during the GeneXus→Java transition). Keep
      SAU_REM.apresentacaoCodigo a raw nullable id (NO @ManyToOne / NO FK constraint) and treat 0 as
      'não informado'. Parity write-scenarios must seed their own SAU_APRREM rows.
  - id: OQ-6
    topic: business-rule
    status: RESOLVED 2026-06-21
    resolution: "Keep the artificial 6-digit code cap (R6: AprRemCod 0..999999). Sequence still used for generation; the cap is validated on input."
```

---

## Domain notes

**SAU_APRREM** is a tiny single-table reference lookup (3 columns) feeding the `AprRemCod` FK on
`SAU_REM`. It is the last Wave-3 dependency partner for the medication catalog. No BC, no
sub-tables, no PHI. The CRUD shape and rule set mirror the other leaf lookups (e.g. SAU_TIPREM):
required descrição/abreviação, UPPERCASE storage, sequence-generated code, delete-guard against
the referrer, audit on write.

**Rule count by confidence:** high 7 (R1-R5, R7, R8, R11) · medium 5 (R6, R9, R10, R12) · total **12**.

**Reference slice to copy when migrating:** `tipo-medicamento` (SAU_TIPREM — same leaf-lookup
shape: code + descrição, sequence, delete-guard) plus the `@JdbcTypeCode(Types.CHAR)` pattern from
`medicamento/Medicamento.java` for the `AprRemAbr` char(5) column.
