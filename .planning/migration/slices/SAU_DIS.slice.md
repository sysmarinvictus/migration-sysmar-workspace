# SLICE-SPEC — SAU_DIS (Distrito Sanitário)

> District office catalog — geographic/administrative reference data. No PHI.
> SAU_DIS **is** a physical DB table (unlike SAU_PACPRN). Already fully defined in
> `V1__baseline.sql` (lines 171-185) — no DDL migration needed for Phase 1.

```yaml
slice: SAU_DIS
domain: distrito
description: Distrito Sanitário — administrative sanitary district with office address and contacts
wave: 2
complexity: M           # sau_dis_impl.java ~162 KB / 3 443 lines
primary_table: SAU_DIS

constellation:
  transaction: sau_dis.java        # 48 lines; WebServlet stub
  impl: sau_dis_impl.java          # 162 KB / 3 443 lines — rule source
  bc: null                         # not a GeneXus Business Component (no gxmetadata/SAU_DIS.xml)
  workwith: null                   # no hwwsau_dis.java
  view: null                       # no hviewsau_dis.java
  prompts: []                      # no hpromptsau_dis*.java
  sections: []
  sdt: null
  helper_procedures:
    - psau_inc_dis.java            # "Gera Código do Distrito" — MAX(DisCod)+1 on INSERT

# SAU_TIPLOG is not in BACKLOG (may be a Wave-1/2 lookup not yet inventoried).
# Keep DisTipLogCod as raw Integer until that slice is clarified.
# SAU_UNI is Wave-2 and only appears via the DELETE guard (not a FK FROM SAU_DIS TO SAU_UNI).
depends_on:
  - SAU_BAI      # DisBaiCod → BaiCod; SAU_BAI status: tested ✓ — FK can be mapped raw Integer
  - SAU_TIPLOG   # DisTipLogCod → TipLogCod; not yet inventoried — raw Integer FK for now

fields:
  # Source: SAU_DISConversion.xml (authoritative DDL) + SAU_DIS.sp0 attri_i + sau_dis_impl.java
  # gxmetadata/SAU_DIS.xml does NOT exist (not a BC); types confirmed via Conversion XML + sp0.
  # All types cross-checked against V1__baseline.sql — full agreement, no discrepancies.

  - { gx: DisCod,        col: DisCod,        name: codigo,               type: Short,   pk: true,  nullable: false,
      note: "Auto-assigned by psau_inc_dis (MAX+1, impl:1860-1872). Read-only in UI. SMALLINT in DB. Replace with DB SEQUENCE in new service — see OQ2." }

  - { gx: DisNom,        col: DisNom,        name: nome,                 type: String,  len: 30,   nullable: true,
      note: "Required (R2). Stored uppercase — server must normalise via toUpperCase() (R4)." }

  - { gx: DisTipLogCod,  col: DisTipLogCod,  name: tipoLogradouroCodigo, type: Integer, nullable: true,
      fk: { ref: SAU_TIPLOG, raw: true },
      note: "Optional FK (0 = null sentinel, impl:1119). Non-zero must exist in SAU_TIPLOG (R5). DisTipLogSig derived read-only from SAU_TIPLOG — NOT a column on SAU_DIS." }

  - { gx: DisEnd,        col: DisEnd,        name: endereco,             type: String,  len: 50,   nullable: true,
      note: "Stored uppercase (R4)." }

  - { gx: DisNum,        col: DisNum,        name: numero,               type: Short,   nullable: true,
      note: "Range 0-9999 (R21); SMALLINT in DB." }

  - { gx: DisCom,        col: DisCom,        name: complemento,          type: String,  len: 15,   nullable: true,
      note: "Stored uppercase (R4)." }

  - { gx: DisBaiCod,     col: DisBaiCod,     name: bairroCodigo,         type: Integer, nullable: true,
      fk: { ref: SAU_BAI, raw: true },
      note: "Optional FK (0 = null sentinel, impl:1135). Non-zero must exist in SAU_BAI (R8). DisBaiNom derived read-only from SAU_BAI — NOT a column on SAU_DIS. bai↔dis cycle — keep nullable (see OQ5)." }

  - { gx: DisCEP,        col: DisCEP,        name: cep,                  type: Integer, nullable: true,
      note: "Stored as plain integer; display mask '99.999-999' is UI-only (R20). INTEGER in DB." }

  - { gx: DisDDD,        col: DisDDD,        name: ddd,                  type: String,  len: 3,    nullable: true,
      note: "Digits-only when non-blank (R11). VARCHAR(3) in DB." }

  - { gx: DisFon,        col: DisFon,        name: telefone,             type: Integer, nullable: true,
      note: "Local phone number stored as integer; display mask '9999-9999' is UI-only. DDD stored separately (R20)." }

  - { gx: DisFax,        col: DisFax,        name: fax,                  type: Integer, nullable: true,
      note: "Same as DisFon (R20)." }

  # DERIVED display fields — NOT columns on SAU_DIS; fetched by the service via lookup query:
  #   DisTipLogSig (from SAU_TIPLOG.TipLogSig) — include in response DTO only
  #   DisBaiNom    (from SAU_BAI.BaiNom)        — include in response DTO only

keys:
  pk: [DisCod]
  unique: []
  fks:
    - { cols: [DisTipLogCod], ref: SAU_TIPLOG, nullable: true,
        note: "0 is sentinel for 'not provided'; exempted from existence check (impl:1119)" }
    - { cols: [DisBaiCod],    ref: SAU_BAI,    nullable: true,
        note: "bai↔dis cycle — must remain soft FK during Wave 2 (BACKLOG.md)" }
  referenced_by:
    - { table: SAU_UNI, col: UniDisCod, note: "DELETE blocked when any SAU_UNI row references this DisCod (R16, impl:1748-1758)" }

endpoints:
  - { method: GET,    path: /api/distritos,       role: SAUDE_CADASTRO,
      from: "T000218 scan (impl:3058)",      desc: "list all, ordered by DisCod; include derived tiplogSigla + bairroNome" }
  - { method: GET,    path: /api/distritos/{id},  role: SAUDE_CADASTRO,
      from: "T00023/T00026 (impl:3043,3046)", desc: "get by DisCod with joined TipLog + Bai names" }
  - { method: POST,   path: /api/distritos,        role: SAUDE_CADASTRO,
      from: "T000212 INSERT (impl:3052)",    desc: "insert; DisCod auto-assigned (R13)" }
  - { method: PUT,    path: /api/distritos/{id},   role: SAUDE_CADASTRO,
      from: "T000213 UPDATE (impl:3053)" }
  - { method: DELETE, path: /api/distritos/{id},   role: SAUDE_CADASTRO,
      from: "T000214 DELETE (impl:3054)",    desc: "blocked when SAU_UNI references this distrito (R16)" }
  # No lookup endpoint (no hprompt file); add GET /api/distritos/lookup if SAU_UNI or
  # SAU_BAI forms need autocomplete when those slices are built.

phi_fields: []
# SAU_DIS is a geographic/institutional catalog (district office contacts).
# Addresses and phone numbers belong to the district office, not any individual.
# No LGPD Art. 5/11 personal data present.

auth:
  roles_required: [SAUDE_CADASTRO]
  notes: >
    GeneXus IntegratedSecurity disabled (impl:1589-1601). Authorization via pisauthorized("SAU_DIS",
    Gx_mode) in Start event (impl:937). Role SAUDE_CADASTRO is conventional — confirm the exact
    module constant against SAU_USUCON / SAU_PRFCON in the live DB (see OQ1).

parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios:
    - list all (GET /api/distritos)
    - get by id — existing DisCod
    - get by id — 404 (non-existent DisCod)
    - insert valid (all fields)
    - insert minimal (DisCod=auto + DisNom only; FK codes = 0)
    - insert duplicate DisCod — 409 Conflict
    - insert with invalid DisTipLogCod (non-zero, non-existent) — 422
    - insert with DisTipLogCod=0 — accepted (null sentinel)
    - insert with invalid DisBaiCod (non-zero, non-existent) — 422
    - insert with non-digit DisDDD — 422
    - update nome
    - update bairroCodigo (valid code)
    - delete unused distrito — 204
    - delete distrito referenced by SAU_UNI — 409

status: verified       # parity 13/13 PARITY (+1 N/A) 2026-06-22. Report: backend/src/test/resources/parity/distrito/PARITY-REPORT.md.
# Entity↔live types all match (no mismatch). #6 duplicate-DisCod N/A (codigo auto-assigned MAX+1 — accepted improvement).
# #9 invalid DisBaiCod rejects via 400 (Bean Validation) not 422 — rejects as legacy (status nuance, RF6). No business divergences.
```

## Mined rules (gx-rule-miner — 22 rules; all sources: sau_dis_impl.java unless noted)

- **R1** — *Authorization via pisauthorized.* GX IntegratedSecurity disabled; `pisauthorized("SAU_DIS", Gx_mode)` called in Start event (impl:937); unauthenticated → `hnotauthorizeduser`, unauthorized → `hnotauthorizedprompt`. New service: Spring Security JWT + `@PreAuthorize`. `kind: authorization` · `confidence: high` · `target: DistritoControllerIT#requiresAuthentication`

- **R2** — *DisNom required on INSERT.* Empty/blank DisNom rejected: "Informe o Nome do Distrito" bound to field DISNOM (impl:1106-1112, `checkExtendedTable0214`). `kind: validation` · `confidence: high` · `target: DistritoServiceTest#rejectsBlankNomeOnInsert`

- **R3** — *DisNom required on UPDATE.* Same `checkExtendedTable0214` check fires before update (impl:1622). `kind: validation` · `confidence: high` · `target: DistritoServiceTest#rejectsBlankNomeOnUpdate`

- **R4** — *Uppercase normalisation.* DisNom, DisEnd, DisCom stored as uppercase — `GXutil.upper()` applied in POST handler (impl:545-546, 568-570, 588-590); `@!` picture and `onblur toUpperCase` on client. Service must normalise before persisting. `kind: derivation` · `confidence: high` · `target: DistritoServiceTest#nomeStoredAsUppercase`

- **R5** — *DisTipLogCod FK check on INSERT.* Non-zero value must exist in SAU_TIPLOG (cursor T00024, impl:1113-1128); rejected with "Não existe 'SAU_DIS_LOG'." Zero is exempt. `kind: referential` · `confidence: high` · `target: DistritoServiceTest#rejectsUnknownTipLogOnInsert`

- **R6** — *DisTipLogCod FK check on UPDATE.* Same check, called from update0214 (impl:1622). `kind: referential` · `confidence: high` · `target: DistritoServiceTest#rejectsUnknownTipLogOnUpdate`

- **R7** — *DisTipLogSig derived read-only.* Populated from SAU_TIPLOG.TipLogSig on any load (impl:1125-1127, cursor T00024); `edtDisTipLogSig_Enabled=0` always (impl:2267). Include in response DTO only — never accept in request body. `kind: derivation` · `confidence: high` · `target: DistritoServiceTest#derivesTipLogSigFromLookup`

- **R8** — *DisBaiCod FK check on INSERT.* Non-zero value must exist in SAU_BAI (cursor T00025, impl:1129-1144); rejected with "Não existe 'Bairro'." Zero is exempt. `kind: referential` · `confidence: high` · `target: DistritoServiceTest#rejectsUnknownBairroOnInsert`

- **R9** — *DisBaiCod FK check on UPDATE.* Same check, called from update0214 (impl:1622). `kind: referential` · `confidence: high` · `target: DistritoServiceTest#rejectsUnknownBairroOnUpdate`

- **R10** — *DisBaiNom derived read-only.* Populated from SAU_BAI.BaiNom on any load (impl:1141-1143); `edtDisBaiNom_Enabled=0` always (impl:2255). Response DTO only. `kind: derivation` · `confidence: high` · `target: DistritoServiceTest#derivesBairroNomeFromLookup`

- **R11** — *DisDDD digits-only on INSERT.* Non-blank DisDDD validated by `GxRegex.IsMatch(A274DisDDD,"[0-9]")` (impl:1145-1151); rejected with "Informe apenas Números!". Blank passes. New service: `@Pattern(regexp="^\\d*$")`. `kind: validation` · `confidence: high` · `target: DistritoServiceTest#rejectsNonDigitDddOnInsert`

- **R12** — *DisDDD digits-only on UPDATE.* Same check, called from update0214. `kind: validation` · `confidence: high` · `target: DistritoServiceTest#rejectsNonDigitDddOnUpdate`

- **R13** — *DisCod auto-assigned on INSERT.* `psau_inc_dis` computes `MAX(DisCod)+1` (or 1 if empty) in `afterConfirm0214` (impl:1860-1872, psau_inc_dis.java:52-70). Field is never editable. **New service must use a DB SEQUENCE** to avoid race condition — OQ2. Effective range 1-127 (SMALLINT). `kind: derivation` · `confidence: high` · `target: DistritoServiceTest#codigoIsAutoAssignedOnInsert`

- **R14** — *Duplicate PK rejected on INSERT.* INSERT returning status=1 (UK violation) triggers `GXM_noupdate` (impl:1584-1588). Safety net under the auto-sequence — still required in service layer. `kind: validation` · `confidence: high` · `target: DistritoServiceTest#rejectsDuplicateCodigo`

- **R15** — *Optimistic concurrency on UPDATE/DELETE.* Re-reads row `FOR UPDATE NOWAIT` (cursor T00022, impl:1476-1559): row locked → `GXM_lock`; any of the 10 mutable columns changed since load → `GXM_waschg`. REST: return 409 on conflict. `kind: validation` · `confidence: high` · `target: DistritoServiceTest#rejectsConcurrentModification`

- **R16** — *DELETE blocked by SAU_UNI.* Cursor T000217 (`SELECT UniCod FROM SAU_UNI WHERE UniDisCod = ? LIMIT 1`): any result blocks delete with `GXM_del("Unidade de Atendimento")` (impl:1748-1758). No SAU_BAI delete guard in this transaction — see OQ5. `kind: referential` · `confidence: high` · `target: DistritoServiceTest#rejectsDeleteWhenReferencedByUnidade`

- **R17** — *Audit on every successful write.* `psau_inc_log` called after commit when `AnyError==0` (impl:1780-1811): EmpCod, timestamp, UsuCod, mode (INS/UPD/DLT), "SAU_DIS", DisCod as key. New service: `common/audit`. `kind: audit` · `confidence: high` · `target: DistritoServiceTest#emitsAuditOnEveryWrite`

- **R18** — *PK immutable on UPDATE.* If in-flight DisCod ≠ snapshot Z32DisCod, rejected with `GXM_getbeforeupd` (impl:1381-1388). REST: path param is authoritative; service must not allow updating a different row. `kind: validation` · `confidence: high` · `target: DistritoServiceTest#rejectsPrimaryKeyChangeOnUpdate`

- **R19** — *Record-not-found on GET.* Non-INS mode with non-existent DisCod raises `GXM_noinsert` (impl:775-779). REST: 404. `kind: validation` · `confidence: high` · `target: DistritoServiceTest#returnsNotFoundForUnknownCodigo`

- **R20** — *CEP/Fone/Fax stored as integers.* DisCEP, DisFon, DisFax are plain integers in the DB; display masks (`99.999-999`, `9999-9999`) are UI-only HMask controls. Service accepts raw integers; DTOs may include formatted strings for display. `kind: format` · `confidence: high` · `target: DistritoServiceTest#cepStoredAsInteger`

- **R21** — *Range guards on numeric FKs.* DisTipLogCod: 0-999999; DisBaiCod: 0-999999; DisNum: 0-9999 (impl:548-607). Out-of-range values rejected with `GXM_badnum` before FK checks fire. `kind: validation` · `confidence: medium` · `target: DistritoServiceTest#rejectsOutOfRangeFields`

- **R22** — *FK direction confirmed.* SAU_DIS holds `DisBaiCod` → SAU_BAI (dis→bai, not the reverse). No FK from SAU_BAI to SAU_DIS exists in this transaction. The BACKLOG "bai↔dis cycle" may originate from SAU_BAI having a `BaiDisCod` column — verify in sau_bai_impl.java (OQ5). `kind: referential` · `confidence: high` · `target: (no test — structural clarification)`

## Schema notes (gx-schema-mapper)

- **SAU_DIS is already in `V1__baseline.sql`** (lines 171-185) — all 11 columns, correct types, PK. **No DDL migration needed.**
- Sources cross-checked: `SAU_DISConversion.xml` + `SAU_DIS.sp0` + baseline — **full agreement, zero discrepancies.**
- `DisCod` and `DisNum` → `Short` (SMALLINT); all FK/CEP/phone fields → `Integer` (INTEGER); varchar fields per lengths above.
- **FK constraints are missing from the baseline** (`DisBaiCod` → SAU_BAI, `DisTipLogCod` → SAU_TIPLOG). Live DB has them (GX DDL confirms). Testcontainer tests seeding invalid FK values will behave differently from production. Append these to baseline when both tables are confirmed (see schema mapper output).
- Both FK columns stay as raw `Integer` (no `@ManyToOne`): DisBaiCod is same-wave bai↔dis cycle; DisTipLogCod is an unmapped lookup.

## Open questions (resolve via KB/live-DB before `/migrate-slice SAU_DIS`)

1. **OQ1 — Exact role/permission constant.** `pisauthorized("SAU_DIS", mode)` called at impl:937. Query `SAU_PRFCON` or `SAU_USUCON` in the live DB to find the exact permission name bound to this module. `SAUDE_CADASTRO` is the conventional placeholder.

2. **OQ2 — DisCod sequencing strategy.** `psau_inc_dis` uses `MAX(DisCod)+1` (race-prone under concurrent inserts, psau_inc_dis.java:52-70). Does the live DB have a `SEQUENCE` or `SERIAL` on DisCod? SMALLINT range caps at 127 — confirm the current max value with the DBA. New service should use a `CREATE SEQUENCE` or `nextval()`. If close to 127, escalate to the team before migration.

3. **OQ3 — `psau_inc_log` audit signature.** Confirm which table `psau_inc_log` writes to (SAU_LOG or a separate table), and whether it records old values on UPDATE. The `common/audit` framework must replicate this exactly.

4. **OQ4 — DisFon/DisFax semantics.** Are DisFon/DisFax the local number only (DDD stored separately in DisDDD), or does the district sometimes store the full number including DDD in DisFon? Verify with domain lead before designing the response DTO phone display format.

6. **OQ7 — R15 Optimistic concurrency deferred.** SAU_DIS lacks a version/timestamp column; implementing `@Version` (JPA optimistic locking) requires adding a column via Flyway migration and coordinating with the live DB. Deferred until Wave 2 coordinator approves schema-additive migrations. `@Disabled` stub exists in `DistritoServiceTest#rejectsConcurrentModification`.

~~**OQ5 — SAU_BAI back-reference (bai↔dis cycle) — RESOLVED.**~~ `Bairro.java` Javadoc confirms "referenced by SAU_DIS"; `sau_bai_impl.java` has no `BaiDisCod` column. The FK is one-directional: SAU_DIS.DisBaiCod → SAU_BAI. The BACKLOG "bai↔dis cycle" refers only to migration wave ordering (both Wave 2), not a true bidirectional FK. No delete guard needed in the Bairro service for SAU_DIS rows.

6. **OQ6 — SAU_TIPLOG inventory.** SAU_TIPLOG (Tipo de Logradouro) is referenced as a FK but does not appear in the current BACKLOG. Determine whether it needs its own slice or can be covered by a simple stub/seed. Until resolved, `DisTipLogCod` remains a raw Integer with a service-level lookup query.
