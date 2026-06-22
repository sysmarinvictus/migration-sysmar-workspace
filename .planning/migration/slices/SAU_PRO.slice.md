# SLICE-SPEC — SAU_PRO (Profissionais)

> **Wave-4 keystone.** The health-professional / **prescriber** registry. SAU_PRO is a GeneXus
> *subtype/extension transaction over SYS_PES*: its PK `propescod` **is** the SYS_PES person code, so
> name/CPF/address/phones live in SYS_PES and are read (and written back) via the person record.
> Only 16 columns are physically stored in SAU_PRO. This slice is **security- and PHI-critical** — it
> holds the prescriber's CNS, council registration, digital **signing certificate**, signature image,
> and (legacy) a **plaintext certificate password**. It is the prescriber referenced by SAU_RECESP
> (controlled substances, Portaria 344/98). Extracted 2026-06-22 (gx-extractor + gx-rule-miner +
> gx-schema-mapper; schema verified against live `saude-mandaguari`).

```yaml
slice: SAU_PRO
domain: profissional
description: Profissionais
wave: 4
complexity: L            # sau_pro_impl.java 551 KB (BACKLOG: 538). 32 mined rules.
primary_table: SAU_PRO
constellation:
  transaction:   sau_pro.java              # 1.1 KB servlet stub
  impl:          sau_pro_impl.java         # 551 KB — primary rule source
  bc:            null                      # NOT a Business Component; no gxmetadata/sau_pro.json → DB introspection authoritative
  workwith:      hwwsau_pro.java           # + hwwsau_pro_impl.java (144 KB) — list/search/filter grid
  view:          hviewsau_pro.java         # + hviewsau_pro_impl.java (55 KB) — read-only detail
  prompts:       [hpromptsau_pro.java]     # + _impl (104 KB) — FK lookup popup (prescriber selector for SAU_RECESP)
  sections:      [hsau_progeneral.java]    # + _impl (69 KB) — person "general" panel (renders SYS_PES)
  sdt:           null
  procedures:
    - psau_soundex.java        # ProPesNomSoundex phonetic key (ordered ~50-entry replacement table)
    - psau_val_cns.java        # CNS check-digit/format validation (FlagCns 1/2/3)
    - psau_ver_cns.java        # CNS cross-person uniqueness
    - psau_val_cnpjcpf.java    # CPF/CNPJ check-digit validation
    - psau_ver_cnpjcpf.java    # CPF/CNPJ cross-person uniqueness
    - pisauthorized.java       # authorization (superuser bypass + per-mode permission bits)
    - psau_inc_log.java        # audit → SAU_LOG (NOTE: hardcodes logProPesCod=0 — defect, see R18)

# Dependency status (migrate-order / FK handling):
depends_on:
  - { table: SAU_CONCLA, role: "conselho de classe (FK conclacod)", migrated: true }     # ✅
  - { table: SYS_PES,    role: "person supertype — name/CPF/address/phones; PK=FK", migrated: false, wave: 0 }
  - { table: SAU_BAI,    role: "bairro via SYS_PES address (display)", migrated: true }
  - { table: SYS_MUN,    role: "município via SYS_PES (display)", migrated: false }
  - { table: SAU_PROESP, role: "delete-guard: pro has specialties (R20)", migrated: false, wave: 4 }
  - { table: SAU_RECESP, role: "delete-guard: pro has controlled prescriptions (R26, Portaria 344)", migrated: false }
  - { table: SAU_USU,    role: "delete-guard: pro linked to a system user (R21)", migrated: false, wave: 0 }
  - { table: SAU_UNI,    role: "delete-guards: 4 unidade roles — autorizador/auditor/diretor/médico (R25)", migrated: true }
  # delete-guard tables outside the backlog waves (implement as native count(*) checks; confirm physical names — OQ-DG):
  - { table: "uni-nut-pro-pes", role: "delete-guard R22 (T000A43)", migrated: false }
  - { table: SISPRENATAL, role: "delete-guard R23 (T000A44)", migrated: false }
  - { table: HIPERDIA,   role: "delete-guard R24 (T000A45)", migrated: false }

# ── OWN COLUMNS (persisted in SAU_PRO) — 16, authoritative from live DB + INSERT/UPDATE SQL ──
fields:
  - { gx: ProPesCod,            name: id,                  type: long,    pk: true,  nullable: false, stored: true,
      note: "= FK to SYS_PES.PesCod (1:1 supertype; PK IS the FK). User-selected from an existing Pessoa; NOT auto-generated. src sau_pro_impl.java:2161-2169 (person must exist)" }
  - { gx: ProPesNumCns,         name: numeroCns,           type: string, len: 20, nullable: true, stored: true, phi: true,
      note: "CHAR(20) → @JdbcTypeCode(CHAR). Required (R3); validated 15-digit CNS (R4); unique across people (R5)." }
  - { gx: ProPesNomSoundex,     name: nomeSoundex,         type: string, len: 50, nullable: true, stored: true, derived: true, readonly: true,
      note: "varchar(50). Phonetic key COMPUTED from SYS_PES.PesNom via psau_soundex on every confirm (R15). Service recomputes; never client-set." }
  - { gx: ProNumCr,             name: numeroCr,            type: string, len: 20, nullable: true, stored: true, phi: true,
      note: "CHAR(20) → @JdbcTypeCode(CHAR). Council registration nº. No format/required validation in legacy (R14)." }
  - { gx: ProDatIni,            name: dataInicio,          type: date, nullable: true, stored: true, note: "validity start. No cross-validation in legacy (R14)." }
  - { gx: ProDatFim,            name: dataFim,             type: date, nullable: true, stored: true, note: "validity end. No datini<=datfim check in legacy (R14)." }
  - { gx: ProScnesID,           name: cnesId,              type: string, len: 20, nullable: true, stored: true, note: "varchar(20). CNES establishment id." }
  - { gx: ProExpEsus,           name: exportaEsus,         type: boolean, nullable: true, stored: true, note: "real PG boolean. Defaults false on INS (R13)." }
  - { gx: ProExt,               name: externo,             type: short, nullable: true, stored: true,
      note: "smallint → @JdbcTypeCode(SMALLINT). Defaults 0 on INS (R13). Routes After-Trn (0=normal pro / non-zero=extension person) — semantics OQ-EXT (R32)." }
  - { gx: ProSit,               name: situacao,            type: short, nullable: true, stored: true,
      note: "smallint → @JdbcTypeCode(SMALLINT). 1=ATIVO (default on INS, R12) / 2=INATIVO. Live: 3008×1, 6×2. Expose enum in DTO." }
  - { gx: assinaturaImagem,     name: assinaturaImagem,    type: bytea, nullable: true, stored: true, sensitive: true,
      note: "byte[] @JdbcTypeCode(VARBINARY) @Basic(LAZY). Signature image. EXCLUDE from default DTOs; serve only via dedicated audited route." }
  - { gx: assinaturaImagemTipo, name: assinaturaImagemTipo, type: string, len: 3, nullable: true, stored: true,
      note: "CHAR(3) → @JdbcTypeCode(CHAR). Content-type of the signature image." }
  - { gx: ConClaCod,            name: conselhoClasseCod,   type: short, nullable: true, stored: true, fk: { ref: SAU_CONCLA, optional: true },
      note: "smallint. 0 OR null = none (377 rows have 0). Map as raw Short id + lookup (NOT hard @ManyToOne) due to 0-sentinel. Must exist when !=0 (R10)." }
  - { gx: ProUFConselho,        name: ufConselho,          type: string, len: 2, nullable: true, stored: true, note: "varchar(2). UF of the issuing council." }
  - { gx: ProCertificado,       name: certificado,         type: bytea, nullable: true, stored: true, sensitive: true,
      note: "byte[] @JdbcTypeCode(VARBINARY) @Basic(LAZY). ICP-Brasil digital signing certificate. EXCLUDE from DTOs; ACL+audit. Written via deferred blob cursor (R30)." }
  - { gx: ProCertificadoSenha,  name: certificadoSenha,    type: string, len: 50, nullable: true, stored: true, secret: true,
      note: "⚠ varchar(50) PLAINTEXT cert password. @JsonIgnore — NEVER serialize/return/log. Encrypt at rest in modern app. Legacy leaks it (R31) — DO NOT port that." }

# ── DISPLAY-ONLY (from SYS_PES / SAU_CONCLA joins — stored:false, never written to SAU_PRO) ──
fields_display_only:
  - { gx: ProPesNom,     name: nome,            source_table: SYS_PES,    phi: true,  note: "professional name (SAU_PRO has NO name column). SYS_PES.PesNom." }
  - { gx: ProPesCPFCNPJ, name: cpfCnpj,         source_table: SYS_PES,    phi: true,  note: "validated when present (R6); unique across people (R7)." }
  - { gx: ProPesRGIE,    name: rgIe,            source_table: SYS_PES,    phi: true }
  - { gx: ProPesSex,     name: sexo,            source_table: SYS_PES,    phi: true }
  - { gx: ProPesNasDat,  name: dataNascimento,  source_table: SYS_PES,    phi: true }
  - { gx: ProPesEnd,     name: endereco,        source_table: SYS_PES,    phi: true, note: "+ numero/complemento/cep/bairro/municipio" }
  - { gx: ProPesFon,     name: telefone,        source_table: SYS_PES,    phi: true, note: "format-validated when present (R8)" }
  - { gx: ProPesCel,     name: celular,         source_table: SYS_PES,    phi: true, note: "format-validated when present (R9)" }
  - { gx: ConClaNom,     name: conselhoClasseNome,  source_table: SAU_CONCLA, note: "A861ConClaNom" }
  - { gx: ConClaSigra,   name: conselhoClasseSigla, source_table: SAU_CONCLA, note: "A179ConClaSigra (shown in grid)" }
  # NOTE: full SYS_PES person panel (nacionalidade, certidões, título eleitor, CTPS, filiação, etnia,
  #   escolaridade, A803–A847) is rendered read-only by hsau_progeneral — belongs to the SYS_PES (Wave-0)
  #   slice. SAU_PRO INSERT/UPDATE touch ONLY the 16 columns above (+ a name/cpf/phone write-back, R2). OQ7.

keys:
  pk: [ProPesCod]
  unique:
    - { cols: [ProPesNumCns], enforced_by: service, note: "CNS unique across people (R5) — app-enforced, NOT a DB UNIQUE; nullable. Confirm DBA (OQ-CNS-UQ)." }
    - { cols: [ProPesCPFCNPJ], enforced_by: service, note: "CPF/CNPJ unique across people (R7) — SYS_PES-level." }
  indexes: [{ index: isau_pro2, cols: [ConClaCod] }]   # present live
  fks: []   # NO physical FK constraints on the live table (GeneXus enforces in app layer)
  logical_fks:
    - { cols: [ProPesCod], ref: SYS_PES,    refCols: [PesCod],   relation: "1:1 supertype (PK=FK)", migrated: false }
    - { cols: [ConClaCod], ref: SAU_CONCLA, refCols: [ConClaCod], relation: "lookup (0=none)", migrated: true }

endpoints:
  - { method: GET,    path: /api/profissionais,       from: hwwsau_pro,   role: "SAU_PRO:DSP",
      note: "list/search. Filters: id, nome (PesNom LIKE substring — R16), cpfCnpj (=), numeroCns (=), externo, situacao." }
  - { method: GET,    path: /api/profissionais/{id},  from: hviewsau_pro, role: "SAU_PRO:DSP", note: "detail; joins SYS_PES + SAU_CONCLA. NEVER returns certificado/senha." }
  - { method: POST,   path: /api/profissionais,       from: "sau_pro INS", role: "SAU_PRO:INS", note: "person (SYS_PES) must pre-exist (R1). Body excludes id-gen, soundex, certificadoSenha-in-clear policy." }
  - { method: PUT,    path: /api/profissionais/{id},  from: "sau_pro UPD", role: "SAU_PRO:UPD", note: "may write back SYS_PES name/cpf/phones (R2)." }
  - { method: DELETE, path: /api/profissionais/{id},  from: "sau_pro DLT", role: "SAU_PRO:DLT", note: "blocked by 9 delete-guards (R20-R26)." }
  - { method: GET,    path: /api/profissionais/lookup, from: hpromptsau_pro, note: "FK autocomplete (prescriber selector)." }
  - { method: GET,    path: /api/profissionais/{id}/assinatura, from: "assinaturaImagem blob",
      note: "OPTIONAL, AUDITED, ACL-checked. Image bytes only. NO endpoint exposes certificado or certificadoSenha." }

phi_fields: [numeroCns, numeroCr, nome, cpfCnpj, rgIe, dataNascimento, endereco, telefone, celular, sexo]
sensitive_fields:
  - { name: certificadoSenha, level: SECRET,  rules: ["never returned by any endpoint", "@JsonIgnore", "encrypt at rest", "never logged", "currently PLAINTEXT in live DB"] }
  - { name: certificado,      level: SIGNING, rules: ["ACL + audit", "ICP-Brasil signing material", "LAZY, excluded from default DTO"] }
  - { name: assinaturaImagem, level: SIGNING, rules: ["ACL + audit", "LAZY, excluded from default DTO"] }

auth:
  integrated_security: false        # IntegratedSecurityEnabled()=false everywhere
  mechanism: pisauthorized
  permission_key: "SAU_PRO"         # pisauthorized("SAU_PRO", Gx_mode); AV10IsAuthorized==0 → denied (sau_pro_impl.java:1444-1460)
  modes: [INS, UPD, DLT, DSP]
  notes: >
    Superuser (Ususysmar) bypasses all (R28). Per-mode bits: INS→UsuInc, UPD→UsuAlt, DLT→UsuExc,
    DSP→UsuCon (pisauthorized.java). Map to @PreAuthorize roles (proposed SAUDE_PROFISSIONAL_* or the
    coarse SAUDE_CADASTRO used by prior slices) — confirm with Wave-0 auth team (OQ6).

parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment/servlet"
  scenarios:
    - list default (page; compare count + first rows)
    - search by nome (PesNom LIKE substring) / by CNS / by CPF
    - view by id (joined SYS_PES name/cpf + SAU_CONCLA sigla; NO cert/senha in payload)
    - insert: person (SYS_PES) does not exist → reject (R1)
    - insert: missing CNS → reject (R3)
    - insert: invalid CNS → reject (R4)
    - insert: duplicate CNS (other person) → reject (R5)
    - insert: invalid CPF (when present) → reject (R6); duplicate CPF → reject (R7)
    - insert: unknown conselho de classe (conclacod!=0) → reject (R10); conclacod=0 allowed
    - insert: situacao defaults to 1 (ATIVO); proext=0, exportaEsus=false (R12/R13)
    - insert/update: nomeSoundex recomputed server-side (R15)
    - update: writes back SYS_PES name/cpf/phones (R2)
    - delete blocked by each guard: SAU_PROESP (R20), SAU_USU (R21), SAU_UNI roles (R25), SAU_RECESP (R26)
    - delete allowed when unreferenced
    - SECURITY: certificadoSenha never appears in any GET/list/view payload (R31)

status: tested   # PARITY 13/13 (2026-06-22, report: backend/src/test/resources/parity/profissional/PARITY-REPORT.md) but NOT verified —
# verified is GATED on the cutover checklist below (DPO/regulatory/OQ-DG/R28 sign-offs parity can't clear). Per user scope 2026-06-22.
# Parity (new app vs live snapshot + mined rules; legacy WW auth-gated so no headless HTML capture; PHI synthetic; snapshot restored
# exactly sys_pes=83105/sau_pro=3014): 13/13 incl. security PASS (no cert/senha/signature in any payload, R31), R1/R3/R4/R5/R10
# validations, R2 SYS_PES write-back, R15 soundex, R16 filters (42P18 ok), R20+R26 delete-guards block, unreferenced delete 204.
# NOT exercised: R21/R25 guards (not seeded), R22/R23/R24 (non-enforcing stubs), R28 per-mode auth, populated-certificate round-trip.
# backend+frontend+tests generated & green 2026-06-22. Full suite: 512 tests, 0F/0E (86 skipped=parity scaffolds).
# Backend 60 slice tests (Service 24, Security 7, Soundex 10, ControllerIT 19) + frontend tsc clean + 10 vitest.
# migration-reviewer: NO BLOCK; fit for tested. VERIFIED is GATED on the cutover checklist below.
# Native SYS_PES person access (read projection + R2 write-back); cert/senha/signature OUT of v1 API; AES-GCM fail-closed
# converter on certificadoSenha; audit uses real professional id (R18 fixed). 3 defects found+fixed during gen
# (delete-guard to_regclass parse crash; encrypted-senha VARCHAR(50)->255 overflow; test enc-key).
#
# CUTOVER CHECKLIST (must clear before `verified`):
#  1. PARITY: /verify-parity SAU_PRO — 0/19 scenarios captured yet (hard blocker).
#  2. SECURITY/DPO sign-off: OQ1 (cert/senha at-rest scheme + bulk-read PHI audit), OQ2 (cert/signature blob ACL+route).
#  3. REGULATORY sign-off: OQ-CNS (mod-11 CNS algorithm), R26 SAU_RECESP retention (soft-deactivate, never hard-delete).
#  4. R22/R23/R24 delete-guards are NON-ENFORCING stubs (to_regclass/query_to_xml; tables absent in live) — confirm
#     real physical table names (OQ-DG) + replace with EXISTS guards + seeded IT, else a referenced prescriber is deletable.
#  5. R28 per-mode auth + superuser bypass (OQ6) — currently coarse hasRole('SAUDE_CADASTRO') like reference slices.
#  6. Flyway: SAU_PRO completed in-place in V1 — confirm no env applied the partial 4-col V1 (checksum), else flyway repair.
status_was: specced
```

## Resolved decisions (2026-06-22)

- **SYS_PES access = native projection (no SYS_PES entity).** Person fields (nome, cpfCnpj, rg, sexo,
  nascimento, endereço, telefone, celular) are READ via native query joining `SYS_PES WHERE PesCod = ProPesCod`.
  The R2 write-back (PesNom/PesCPFCNPJ/PesFon/PesCel) is a native UPDATE against SYS_PES. No `@OneToOne`/
  `@MapsId` — the PK is a raw `Long`. Swap to a JPA relationship only when SYS_PES (Wave 0) is migrated.
- **conclacod** = raw `Short` column + `ConselhoClasseRepository` lookup (0/null = none); NOT `@ManyToOne`.
- **Inherited SYS_PES validations (R6–R9, R11)** operate on person fields → implement in the service over
  the projected/submitted person sub-DTO; CPF/CNS/phone validators ported as Spring validators.
- **Security (non-negotiable):** `certificadoSenha` → `@JsonIgnore`, encrypted at rest (envelope/Jasypt or
  a `@Convert` AES converter keyed from env), NEVER logged/returned; `certificado`/`assinaturaImagem` LAZY,
  excluded from list/detail DTOs, served only via a dedicated audited route (deferred — not in v1 DTOs).
  Do NOT port R31's cleartext logging. Audit records the REAL professional id (fix R18).
- **Status ceiling:** generation may reach `tested`; `verified` stays BLOCKED on OQ1/OQ2/OQ-CNS
  (DPO/security + regulatory sign-off) and on SAU_RECESP retention semantics (R26 soft-deactivate).

## Mined rules

> 32 rules from `sau_pro_impl.java` (+ `psau_soundex`, `pisauthorized`, `psau_inc_log`, validation
> procs). Exact `file:line` cites; confidence high=explicit validation/message, medium=inferred.
> Full rule bodies retained from gx-rule-miner — see citations.

### PK / person linkage
- **R1** (insert|update, validation, high) — `propescod` **is** the SYS_PES person code, **user-entered, NOT auto-generated** (only prompt on the form is for ConClaCod). The referenced person must already exist in SYS_PES (cursor T000A6; 101 → "Não existe Profissional"/ForeignKeyNotFound). SAU_PRO INS turns an existing Pessoa into a Profissional; it never auto-creates SYS_PES. src `sau_pro_impl.java:2161-2169, 1684, 4157-4162`. → `ProfissionalServiceTest#rejectsCreateWhenPersonDoesNotExist`.
- **R2** (insert|update, side-effect, high) — on confirm, writes back to SYS_PES the edited PesNom, PesCPFCNPJ, PesFon, PesCel (cursor T000A51). Editing a professional can mutate the underlying person. src `sau_pro_impl.java:4157-4162`. → `#updatesUnderlyingPessoaNameCpfPhones`.

### Required fields & identifier validation
- **R3** (validation, high) — CNS required: blank → "Informe o Número do CNS!". src `:2447-2453`. → `#requiresCns`.
- **R4** (validation, medium) — CNS check-digit/format via `psau_val_cns` (FlagCns 1=inválido / 2=somente números / 3=15 dígitos). Stored char(20) but rule enforces 15 digits. **Regulatory weight** (prescriber id on Portaria-344 prescriptions) — verify exact mod-11 algorithm with regulatory lead. src `:2454-2461, 2140-2160` + `psau_val_cns.java`. → `#rejectsInvalidCns`.
- **R5** (validation, high) — CNS uniqueness across people via `psau_ver_cns` (excludes current id): duplicate → "Este número de CNS está sendo utilizado pelo cadastro <id>!". src `:2462-2480`. → `#rejectsDuplicateCnsAcrossPeople`.
- **R6** (validation, high) — CPF/CNPJ validated when non-empty via `psau_val_cnpjcpf` → "CPF inválido!". CPF NOT required. src `:2397-2413`. → `#rejectsInvalidCpfWhenPresent`.
- **R7** (validation, high) — CPF/CNPJ uniqueness across people (`psau_ver_cnpjcpf`, excludes current) → "Este número de CPF está sendo utilizado pelo cadastro <id>!". src `:2414-2432`. → `#rejectsDuplicateCpfAcrossPeople`.
- **R8** (validation, high) — phone, if present, matches `^(\([0-9]{2}\))\s([9]{1})?([0-9]{4})-([0-9]{4})$` else "Telefone inválido!". src `:2433-2439`. → `#rejectsInvalidPhone`.
- **R9** (validation, high) — mobile, same regex, else "Celular inválido!". src `:2440-2446`. → `#rejectsInvalidMobile`.

### FK existence
- **R10** (validation, high) — ConClaCod must exist in SAU_CONCLA when !=0 (T000A4; 101 → "Não existe Conselho de Classe."). 0 allowed (optional). src `:2481-2492`. → `#rejectsUnknownConselhoDeClasse`.
- **R11** (validation, high) — inherited SYS_PES extended-table FK checks (TipLog, Mun, NacMun, Bai, Etn, Pais, OrgEmi), each ForeignKeyNotFound when code!=0 and missing. **No CBO and no SAU_PROESP FK check inside the SAU_PRO transaction itself.** src `:2293-2389`. → `#rejectsUnknownPersonLookups`.

### Defaults & derivations
- **R12** (default, high) — `prosit` defaults to **1 (ATIVO)** on INS (2=INATIVO). src `:1719-1724, 357-367`. → `#defaultsSituacaoToAtivoOnInsert`.
- **R13** (default, high) — `proext` defaults 0, `proexpesus` defaults false on INS. src `:1725-1736`. → `#defaultsProExtZeroAndExpEsusFalse`.
- **R14** (derivation, high) — `prodatini`/`prodatfim`/`pronumcr` have **no defaults and no validation** (no datini<=datfim, no required, no CR format). Free inputs. **Note:** business may WANT an active-window/prescriber-eligibility rule; legacy doesn't enforce it (product/regulatory decision — OQ-DATE). src `:4871-4876, 6329-6330, 4868, 6326`. → `#dateWindowAndCrAreUnvalidated`.
- **R15** (formula, medium) — `propesnomsoundex` recomputed on every confirm from PesNom via `psau_soundex` (ordered ~50-entry Portuguese phonetic replacement table; order matters). Display-only on the grid as "Nome". **Port the replacement list verbatim** for search parity. src `:4264-4275` + `psau_soundex.java:48-100`. → `#computesSoundexFromName`.
- **R16** (derivation, medium) — list/search (hwwsau_pro) filters: ProPesCod (exact), **ProPesNom via `T3.PesNom LIKE %?%` (substring on the person name, NOT the soundex column)**, CPFCNPJ (=), NumCns (=), ProExt, ProSit. The soundex grid column is display-only, no WHERE. **This reconciles SAU_IMP OQ2: legacy name search is literal LIKE, not phonetic.** src `hwwsau_pro_impl.java:2493-2520`. → `ProfissionalListServiceTest#searchesByPersonNameSubstring`.

### Audit
- **R17** (audit, high) — after every successful write, `psau_inc_log` inserts SAU_LOG (LogOpe=mode, LogTab="SAU_PRO", LogKey=ProPesCod, LogUsuCod, LogDat). src `:4182-4215` + `psau_inc_log.java:89-102,185`. → `#writesAuditRowOnEachWrite`.
- **R18** (audit, high) — **carry-forward-correctly DEFECT:** `psau_inc_log` hardcodes `logProPesCod=0` (and logNomeProfissional='') even though the audited entity IS a professional. Modern audit must record the real professional id (same defect class as SAU_IMP OQ10). src `psau_inc_log.java:185`. → `ProfissionalAuditTest#auditCapturesProfessionalId`.

### Delete guards (onDeleteControls0A4 — 9 referential checks, all "CannotDeleteReferencedRecord"/GXM_del)
- **R19** (delete, high) — umbrella: delete blocked if referenced anywhere. src `:4072-4153`. → `#blocksDeleteWhenReferenced`.
- **R20** (high) — SAU_PROESP (especialidade do profissional), T000A41. src `:4074-4081`. → `#blocksDeleteWithSpecialty`.
- **R21** (high) — SAU_USU (system user), T000A42. src `:4082-4089`. → `#blocksDeleteWithSystemUser`.
- **R22** (medium) — "Uni Nut Pro Pes", T000A43. src `:4090-4097`. → `#blocksDeleteWithUniNut`.
- **R23** (medium) — SISPRENATAL, T000A44. src `:4098-4105`. → `#blocksDeleteWithSisprenatal`.
- **R24** (medium) — HIPERDIA, T000A45. src `:4106-4113`. → `#blocksDeleteWithHiperdia`.
- **R25** (high) — SAU_UNI in 4 roles: Autorizador (T000A46), Auditor (T000A47), Diretor Clínico (T000A48), Médico Responsável (T000A49). src `:4114-4145`. → `#blocksDeleteWhenUnidadeRoleAssigned`.
- **R26** (high) — **SAU_RECESP** (Receituário Controle Especial), T000A50 → a prescriber with any controlled-substance prescription cannot be deleted. **Portaria 344/98 retention**: modern app should never hard-delete a prescriber with SAU_RECESP history (soft-deactivate via situacao). src `:4146-4153`. → `#blocksDeleteWhenHasControlledPrescription`.

### Authorization
- **R27** (always, authorization, high) — Start event: empty Usulogin → redirect unauthenticated; else `pisauthorized("SAU_PRO", Gx_mode)`; AV10IsAuthorized==0 → denied. src `:1430-1460`. → `ProfissionalControllerIT#deniesUnauthenticatedAndUnauthorized`.
- **R28** (always, authorization, high) — `pisauthorized`: **superuser (Ususysmar) bypass**; else per-mode permission bits INS→UsuInc / UPD→UsuAlt / DLT→UsuExc / DSP→UsuCon per (UsuCod, PrgCod='SAU_PRO'). **Security-critical — reproduce exactly.** src `pisauthorized.java:60-67, 98-219`. → `AuthorizationServiceTest#perModePermissionBitsAndSuperuserBypass`.

### Certificate & digital-signature handling (SECURITY-CRITICAL)
- **R29** (insert, side-effect, high) — INSERT writes assinaturaimagem (bytea), procertificado (bytea), procertificadosenha (**cleartext varchar50**) via T000A27; assinaturaimagemtipo derived from upload filetype. No password-strength check. **SECURITY: encrypt at rest in modern app.** src `:3655-3659`. → `#persistsCertificateAndSignatureOnInsert`.
- **R30** (update, side-effect, high) — UPDATE writes senha+tipo inline (T000A28) but the two bytea blobs via **separate deferred cursors** (assinaturaimagem T000A29, procertificado T000A30). src `:3713-3717, 3754-3768`. → `#updatesCertificateBlobsViaDeferredWrite`.
- **R31** (always, audit, high) — **SECURITY DEFECT — DO NOT PORT:** procertificadosenha is round-tripped to the browser as a hidden CGI field AND **written to the app log in cleartext** via `GXutil.writeLogln` on integrity-check failure ("SecurityCheckFailed value for ProCertificadoSenha:<plaintext>"). (Good: absent from view/list/prompt read screens.) Modern app must never log/echo it. src `:1205, 1217, 889/960, 1087-1089`. → `ProfissionalSecurityTest#neverLogsOrEchoesCertificatePassword`.

### Navigation
- **R32** (after_trn, derivation, medium) — After-Trn routes on `proext`: 0 → professional list; non-zero → 'professional-extension' person list. Carry `proext` classification into the domain model (exact values OQ-EXT). src `:1478-1488`. → `#proExtClassification`.

## JPA entity mapping (verified against live DB — gx-schema-mapper)

Target: `br.gov.mandaguari.saude.profissional.domain.Profissional`, `@Entity @Table(name="SAU_PRO")`.
Imports: `org.hibernate.annotations.JdbcTypeCode`, `java.sql.Types`.

| GX col | field | type | annotations |
|--------|-------|------|-------------|
| ProPesCod | id | Long | `@Id @Column(name="ProPesCod", nullable=false)` (raw Long — SYS_PES un-migrated; future @OneToOne @MapsId) |
| ProPesNumCns | numeroCns | String | `@Column(name="ProPesNumCns", length=20) @JdbcTypeCode(Types.CHAR)` |
| ProPesNomSoundex | nomeSoundex | String | `@Column(name="ProPesNomSoundex", length=50)` (service-computed) |
| ProNumCr | numeroCr | String | `@Column(name="ProNumCr", length=20) @JdbcTypeCode(Types.CHAR)` |
| ProDatIni / ProDatFim | dataInicio / dataFim | LocalDate | `@Column(...)` |
| ProScnesID | cnesId | String | `@Column(name="ProScnesId", length=20)` |
| ProExpEsus | exportaEsus | Boolean | `@Column(name="ProExpeSus")` (real boolean) |
| ProExt | externo | Short | `@Column(name="ProExt") @JdbcTypeCode(Types.SMALLINT)` |
| ProSit | situacao | Short | `@Column(name="ProSit") @JdbcTypeCode(Types.SMALLINT)` (1/2 enum in DTO) |
| assinaturaImagem | assinaturaImagem | byte[] | `@Column(name="AssinaturaImagem") @JdbcTypeCode(Types.VARBINARY) @Basic(fetch=LAZY)` |
| assinaturaImagemTipo | assinaturaImagemTipo | String | `@Column(name="AssinaturaImagemTipo", length=3) @JdbcTypeCode(Types.CHAR)` |
| ConClaCod | conselhoClasseCod | Short | `@Column(name="ConClaCod") @JdbcTypeCode(Types.SMALLINT)` (raw id + lookup; 0/null=none — NOT @ManyToOne, due to 0-sentinel) |
| ProUFConselho | ufConselho | String | `@Column(name="ProUfConselho", length=2)` |
| ProCertificado | certificado | byte[] | `@Column(name="ProCertificado") @JdbcTypeCode(Types.VARBINARY) @Basic(fetch=LAZY)` |
| ProCertificadoSenha | certificadoSenha | String | `@Column(name="ProCertificadoSenha", length=50)` **+ `@JsonIgnore`; encrypt at rest; never log** |

**Mapping notes:** PG `bytea` → `@JdbcTypeCode(Types.VARBINARY)` (NOT `@Lob` — `@Lob`+byte[] maps to `oid`/large-object and breaks `validate`). Both blobs LAZY + excluded from default DTOs. `conclacod` 0-sentinel (377 rows) → raw Short + repository lookup, not a hard association. No physical FK constraints on the live table (logical only).

## Flyway plan

- **Production (live):** SAU_PRO exists with all 16 columns; `ddl-auto=validate` → **no ALTER, no forward migration**.
- **Testcontainers V1 baseline:** the current stub has only **4 of 16 columns** (ProPesCod, ConClaCod, ProSit, ProPesNomSoundex) — missing 12; the full entity will **fail `validate`**. Replace the V1 SAU_PRO stub with the complete CREATE TABLE (below) so Testcontainers builds the live shape. (If any test/dev DB already applied V1 — checksum risk — instead add a forward `V__sau_pro_complete_columns.sql` with `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`.)

```sql
CREATE TABLE IF NOT EXISTS SAU_PRO (
    ProPesCod             BIGINT       NOT NULL,
    ProPesNumCns          CHAR(20),
    ProPesNomSoundex      VARCHAR(50),
    ProNumCr              CHAR(20),
    ProDatIni             DATE,
    ProDatFim             DATE,
    ProScnesId            VARCHAR(20),
    ProExpeSus            BOOLEAN,
    ProExt                SMALLINT,
    ProSit                SMALLINT,
    AssinaturaImagem      BYTEA,
    AssinaturaImagemTipo  CHAR(3),
    ConClaCod             SMALLINT,
    ProUfConselho         VARCHAR(2),
    ProCertificado        BYTEA,
    ProCertificadoSenha   VARCHAR(50),
    CONSTRAINT pk_sau_pro PRIMARY KEY (ProPesCod)
);
CREATE INDEX IF NOT EXISTS isau_pro2 ON SAU_PRO (ConClaCod);
```

Verified compatible: SAU_CONCLA exists live + V1 (`ConClaCod smallint` PK ↔ migrated `ConselhoClasse.codigo : Short`). SYS_PES exists live, un-migrated (raw-id PK deferral). `prosit` live values only 1/2.

## Open questions (resolve before `verified`; ⚠ = security/regulatory sign-off)

- **OQ1 ⚠ (certificadoSenha plaintext):** confirm with DPO/security the legacy cleartext storage and define the encryption-at-rest scheme. Must never appear in any DTO/response/log.
- **OQ2 ⚠ (certificado / assinaturaImagem access):** define ACL + audit for reading/replacing the signing certificate and signature image (legal artifacts under Portaria 344). Confirm NO unauthenticated/public blob route (CONCERNS).
- **OQ-CNS ⚠ (R4 algorithm):** transcribe `psau_val_cns.java` exactly (15-digit mod-11) and get regulatory sign-off — this is the prescriber identifier on controlled-substance prescriptions.
- **OQ3 (ProSit domain):** confirm enum values (1=ATIVO/2=INATIVO assumed; live data agrees).
- **OQ-CNS-UQ (CNS uniqueness):** app-enforced only; confirm with DBA whether a DB UNIQUE should be added (nullable).
- **OQ-DATE (R14):** legacy does NOT validate datini<=datfim or prescriber active-window. Add as a NEW rule (product/regulatory decision) or preserve legacy permissiveness?
- **OQ5 (numeroCr):** no format/required validation, no link to ConClaCod/UF — should CR be required/validated per council?
- **OQ6 (auth roles):** map `pisauthorized("SAU_PRO", mode)` + superuser bypass + per-mode bits to concrete `@PreAuthorize` roles via pisauthorized.java / SAU_PRF/SAU_PRFCON/SAU_USUCON.
- **OQ7 (SYS_PES person-panel scope):** SAU_PRO INS/UPD touch only the 16 columns + a name/cpf/phone write-back (R2). Confirm person edits otherwise belong to the SYS_PES (Wave-0) slice; decide whether this slice exposes person editing at all.
- **OQ-EXT (R32 proext):** exact value domain/meaning (0=normal vs non-zero=extension) — observed only via After-Trn routing.
- **OQ-DG (delete-guard tables):** confirm physical table/FK names for the un-backlogged guards — "Uni Nut Pro Pes" (R22), SISPRENATAL (R23), HIPERDIA (R24) — so native count(*) pre-checks match. SAU_PROESP/SAU_RECESP/SAU_USU guards reference un-migrated tables → implement as native existence queries (like SAU_BAI/SAU_IMP) until migrated.
- **OQ8 (SAU_CBOR):** NOT referenced anywhere in the SAU_PRO constellation (no Cbor attribute, absent from TableAccess) — CBO belongs to SAU_PROESP/SAU_ESP, not SAU_PRO. Drop SAU_CBOR from this slice's deps.
- **OQ-SEARCH (R16 vs SAU_IMP OQ2):** legacy SAU_PRO name search is literal `PesNom LIKE`, not phonetic — the soundex column is display-only. Confirm no phonetic-search entry point exists elsewhere; if not, the SAU_IMP LIKE-based filter is actually MORE faithful than its OQ2 feared.
```
