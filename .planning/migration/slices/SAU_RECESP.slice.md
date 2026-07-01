# SLICE-SPEC — SAU_RECESP (Receituário Controle Especial) · Wave-6 · **REGULATORY-GATED**

> ⚠ **Controlled-substance prescription — Portaria SVS/MS 344/98.** This slice MUST NOT be marked
> `verified`/`cutover` until the regulatory open-questions below are signed off by a CRF/regulatory
> lead (CLAUDE.md Rule 5). Extraction only — no code generated yet. Two extractor + schema + rule-miner
> agents cross-checked against the live DB (`host.docker.internal:5432/saude-mandaguari`, 1510 rows) and
> the legacy impl on 2026-07-01.

```yaml
slice: SAU_RECESP
domain: receituario-especial            # NAMING.md:19 → /api/receituarios-especiais
description: Receituário Controle Especial   # sau_recesp.java header
wave: 6
complexity: L                           # BACKLOG.md:107 "L (460)". NB impl=471 KB; effective surface is XL
                                        # (print rsau_recesp 143 KB + _pagint 111 KB + copia 33 KB + child wc 110 KB)
primary_table: SAU_RECESP               # + child SAU_RECESP1 (line items) — this is a MASTER+CHILD aggregate
status: specced

constellation:                          # bytes from ls of Receituario/JavaModel/web/
  transaction: sau_recesp.java                 # 1176
  impl: sau_recesp_impl.java                   # 471780  — RULE SOURCE
  bc: null                                      # NON-BC transaction (no _bc, no SdtSAU_RECESP, no gxmetadata json)
  workwith: [hwwsau_recesp.java, hwwsau_recesp_impl.java]      # 162718 → list/search grid
  view: [hviewsau_recesp.java, hviewsau_recesp_impl.java]     # 63727  → read-only detail
  sections: [hsau_recespgeneral_impl.java]                    # 94174  → general form section
  subform_child: [hsau_recespsau_recesp1wc_impl.java]         # 110243 → SAU_RECESP1 line-items web-component (CORE)
  prompts: []                                   # lookups reuse migrated SAU_PAC/SAU_PRO/SAU_REM prompts
  procs:
    - psau_inc_recesp.java                       # 4356   → SEQUENTIAL NUMBER ALLOCATOR (RecEspCod = max+1 per unit)
    - psau_recesp_copia.java                     # 33237  → "Copiar Receituário" (clone)
    - rsau_recesp_impl.java                      # 143106 → numbered 2-via PRINT report
    - rsau_recesp_pagint_impl.java               # 111461 → alt paginated print
    - receituario_verificainteracaooumpp.java    #        → drug-interaction / MPP advisory (per-unit flag)
    - psau_inc_log.java                          #        → audit writer (→ common/audit)

depends_on:                             # all data-owning parents already migrated → slice may proceed
  - { table: SAU_RECESP1, role: "child detail = prescribed line items", status: part-of-this-slice }
  - { table: SAU_PAC,   role: "patient FK (PacPesCod) — pac↔recesp CYCLE", status: tested }
  - { table: SAU_PRO,   role: "prescriber FK (RecEspProPesCod)",          status: tested }
  - { table: SAU_FUN,   role: "issuer FK (FunPesCod)",                    status: tested }
  - { table: SAU_UNI,   role: "unit FK + PK part (RecEspUniCod)",         status: verified }
  - { table: SAU_REM,   role: "medication FK (RemCod on child)",          status: verified }
  - { table: SAU_REMOBS,role: "posology FK (RemObsCod on child)",         status: verified }
  - { table: SAU_CONCLA,role: "prescriber council/CRF (print only)",      status: verified }
  - { table: SYS_PES,   role: "person supertype (name/CPF via pac/pro)",  status: tested }
  - { table: SAU_PAR,   role: "print/numbering parameters",               status: tested }
  # print-only / advisory deps not modeled as data: SAU_ORGEMI, SYS_EMP, SYS_MUN, SAU_INTERAMED (interaction/MPP)

# ── FIELDS ── live-DB introspected (public.sau_recesp / public.sau_recesp1). NO DB-level FK/unique/check
#    constraints exist (GeneXus enforces in-app) → all parent links are RAW ID columns, no @ManyToOne.
fields:
  # MASTER sau_recesp (header) — 11 cols, PK (RecEspUniCod, RecEspCod)
  - { gx: RecEspUniCod,   name: unidadeCodigo,     type: int,     pk: true,  nullable: false, note: "unit + PK part → SAU_UNI" }
  - { gx: RecEspCod,      name: numero,            type: long,    pk: true,  nullable: false, note: "sequential per-unit prescription number (R1)" }
  - { gx: RecEspDat,      name: data,              type: date,    pk: false, nullable: true,  note: "issue date (required R4)" }
  - { gx: FunPesCod,      name: funcionarioCodigo, type: long,    pk: false, nullable: true,  note: "issuer → SAU_FUN (optional, 0 allowed R9)" }
  - { gx: PacPesCod,      name: pacienteCodigo,    type: long,    pk: false, nullable: true,  note: "patient → SAU_PAC (required R5; **pac↔recesp cycle**)" }
  - { gx: RecEspProPesCod,name: prescritorCodigo,  type: long,    pk: false, nullable: true,  note: "prescriber → SAU_PRO (required R6)" }
  - { gx: RecEspSeqUlt,   name: sequenciaUltima,   type: smallint,pk: false, nullable: true,  note: "last child seq counter (R26); @JdbcTypeCode(SMALLINT)" }
  - { gx: RecEspUsuLogin, name: usuarioLogin,      type: "char(20)", pk: false, nullable: true, note: "audit login, server-stamped (R17); @JdbcTypeCode(CHAR)+trim" }
  - { gx: RecEspCon,      name: situacao,          type: smallint,pk: false, nullable: true,  note: "status/confirm — NO state machine (R30); @JdbcTypeCode(SMALLINT)" }
  - { gx: RecTip,         name: tipoReceituario,   type: smallint,pk: false, nullable: true,  note: "type — all 0 live, meaning unknown Q1; @JdbcTypeCode(SMALLINT)" }
  - { gx: RecObs,         name: observacao,        type: "varchar(300)", pk: false, nullable: true, note: "header notes (≤300 R7)" }
  # CHILD sau_recesp1 (line items = prescribed controlled drugs) — PK (RecEspUniCod, RecEspCod, RecEspSeq)
  - { gx: RecEspSeq,      name: sequencia,         type: smallint,pk: true,  nullable: false, table: SAU_RECESP1, note: "line sequence (R26)" }
  - { gx: RemCod,         name: medicamentoCodigo, type: int,     pk: false, nullable: true,  table: SAU_RECESP1, note: "medication → SAU_REM (optional, 0=free-text R18)" }
  - { gx: RecEspPre,      name: prescricao,        type: "varchar(50)", pk: false, nullable: true, table: SAU_RECESP1, note: "prescription text — REQUIRED per line (R20); defaults from RemNom (R19)" }
  - { gx: RecEspQtd,      name: quantidade,        type: "numeric(5,1)", pk: false, nullable: true, table: SAU_RECESP1, note: "quantity — NOT validated (R24); BigDecimal(5,1)" }
  - { gx: RecEspQtdTip,   name: quantidadeTipo,    type: smallint,pk: false, nullable: true,  table: SAU_RECESP1, note: "enum 1..7 UNIDADE/CAIXA/FRASCO/AMPOLA/COMPRIMIDO/CÁPSULA/ML" }
  - { gx: RemObsCod,      name: posologiaCodigo,   type: int,     pk: false, nullable: true,  table: SAU_RECESP1, note: "posology → SAU_REMOBS (optional R21); defaults RecEspObs (R22)" }
  - { gx: RecEspObs,      name: observacao,        type: "varchar(60)", pk: false, nullable: true, table: SAU_RECESP1, note: "line notes" }
  - { gx: RecEspTip,      name: tipoReceita,       type: smallint,pk: false, nullable: true,  table: SAU_RECESP1, note: "REQUIRED (R23): 1 SIMPLES / 4 SIMPLES-2VIAS / 2 USO-CONTÍNUO-2VIAS / 3 CONTROLE-ESPECIAL-2VIAS" }
  - { gx: RecEspTipUso,   name: tipoUso,           type: smallint,pk: false, nullable: true,  table: SAU_RECESP1, note: "1 USO INTERNO / 2 USO EXTERNO (not enforced R24)" }
  - { gx: RecEspUsoCon,   name: usoContinuo,       type: smallint,pk: false, nullable: true,  table: SAU_RECESP1, note: "continuous-use code (not enforced R24)" }
  - { gx: RecInd,         name: indeferido,        type: boolean, pk: false, nullable: true,  table: SAU_RECESP1, note: "denied flag, native boolean → Boolean; defaults false (R25)" }

keys:
  master: { pk: [RecEspUniCod, RecEspCod], unique: [], fks_logical: [PacPesCod→SAU_PAC, RecEspProPesCod→SAU_PRO, FunPesCod→SAU_FUN, RecEspUniCod→SAU_UNI] }
  child:  { pk: [RecEspUniCod, RecEspCod, RecEspSeq], unique: [], fks_logical: ["(RecEspUniCod,RecEspCod)→SAU_RECESP", RemCod→SAU_REM, RemObsCod→SAU_REMOBS] }
  note: "Composite PKs → @IdClass (precedent ProfissionalEspecialidadeId, usuariounidade/). NO @ManyToOne (no DB FKs; cross-slice raw ids + lookup queries)."

endpoints:                              # {id} = {unidadeCodigo}/{numero}
  - { method: GET,    path: /api/receituarios-especiais,                          from: hwwsau_recesp,  desc: "list/search" }
  - { method: GET,    path: "/api/receituarios-especiais/{unidadeCodigo}/{numero}", from: hviewsau_recesp, desc: "detail incl. line items" }
  - { method: POST,   path: /api/receituarios-especiais,                          from: "sau_recesp INS", desc: "create; server allocates numero via psau_inc_recesp (R1)" }
  - { method: PUT,    path: "/api/receituarios-especiais/{unidadeCodigo}/{numero}", from: "sau_recesp UPD", desc: "update master+items" }
  - { method: DELETE, path: "/api/receituarios-especiais/{unidadeCodigo}/{numero}", from: "sau_recesp DLT", desc: "⚠ legacy HARD-deletes (R29) — modern MUST block/soft-delete pending OQ2" }
  - { method: POST,   path: "/api/receituarios-especiais/{unidadeCodigo}/{numero}/copia",     from: psau_recesp_copia, desc: "clone (R31)" }
  - { method: GET,    path: "/api/receituarios-especiais/{unidadeCodigo}/{numero}/impressao",  from: rsau_recesp, desc: "print numbered 2-via (R32-R35)" }

phi_fields: [pacienteCodigo, prescritorCodigo, funcionarioCodigo, data, situacao, tipoReceituario, observacao,
             medicamentoCodigo, prescricao, quantidade, quantidadeTipo, posologiaCodigo, tipoReceita, tipoUso,
             usoContinuo, indeferido]
# ALL of it: patient identity (resolves to name/CPF/CNS via SAU_PAC/SYS_PES), prescriber identity, and the
# prescribed controlled substance + dosage. LGPD Art.11 sensitive health data + Portaria 344/98.
# Audit EVERY read/write (R28); toString() emits composite id only, never patient/drug detail.

auth:
  roles_required: [SAUDE_CADASTRO]      # coarse role (INFERRED — confirm)
  permission_key: "SAU_RECESP"          # per-program permission passed to pisauthorized (impl:2018), per-mode INS/UPD/DLT/DSP
  notes: >
    Legacy checks pisauthorized("SAU_RECESP", Gx_mode); denies (→ hnotauthorizedprompt) when result==0
    (impl:2006-2033, R3). Per-program RBAC (SAU_RBAC PermissionResolver) is BUILT-BUT-NOT-WIRED, so
    endpoints will initially enforce only the coarse role; wire the "SAU_RECESP" program permission when
    RBAC goes live. Consider a narrower prescribe/emit role distinct from generic cadastro. AcessaModulo.xml
    had no recesp row — verify module mapping in KB.

# ── PERSISTENCE (Flyway) ──
persistence:
  entity: "ReceituarioEspecial (@IdClass ReceituarioEspecialId{unidadeId:Integer, codigo:Long}) + ReceituarioEspecialItem (@IdClass with RecEspSeq)"
  flyway: V14__sau_recesp_receituarioespecial.sql   # next version confirmed (highest existing = V13)
  master_v14: "ADD COLUMN IF NOT EXISTS ×6 (RecEspDat DATE, RecEspSeqUlt SMALLINT, RecEspUsoLogin→RecEspUsuLogin CHAR(20), RecEspCon SMALLINT, RecTip SMALLINT, RecObs VARCHAR(300)) + ALTER RecEspUsuLogin TYPE CHAR(20). All no-op on live; needed for local/Testcontainers ddl-validate. FunPesCod/RecEspProPesCod/PacPesCod already added by V1/V8/V13."
  child_v14: "⚠ ACTION for /migrate-slice: schema-mapper only introspected the MASTER. Re-introspect SAU_RECESP1 and add its missing columns to V14 (baseline stub V1:163-171 + RemCod V5:78-79 exist; still need RecEspPre, RecEspQtd numeric(5,1), RecEspQtdTip, RemObsCod, RecEspObs, RecEspTip, RecEspTipUso, RecEspUsoCon, RecInd)."
  no_hard_schema_change: "Do NOT add a status/cancel column in Phase 1 (live has none) — cancellation modeling is a regulatory decision (OQ2). Do NOT add DB FK/unique constraints."

parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios:
    - "list default page / search by patient"
    - "view by {unidadeCodigo}/{numero} incl. line items"
    - "create valid (verify server-allocated numero = max+1 per unit, R1)"
    - "create rejects: missing date (R4), missing patient (R5), missing prescriber (R6), inactive patient PacSit=2 (R12), missing line prescription text (R20), missing line type (R23)"
    - "create warns-not-blocks: patient without CNS (R13), drug interaction when unit flag on (R27)"
    - "line defaults: RecEspPre←RemNom (R19), RecEspObs←posology (R22)"
    - "audit row written on each write (R28)"
    - "print: type-3 titled 'RECEITUÁRIO CONTROLE ESPECIAL', 1ª/2ª via (R32/R33), council stamp (R34)"
    - "delete behavior — CAPTURE legacy hard-delete (R29) but DO NOT cut over until OQ2 resolved"
  note: "MunCod=411420. Never commit real PHI into fixtures — synthesize/redact patient, prescriber, CPF, CNS."
```

## Rules (mined — 36; confidence tagged; each has a target test)

Full list persisted from `gx-rule-miner`. Summary by confidence: **high 18, medium 13, low 2** (+3 print/regulatory flags carry `_TODO_verify` tests). Key groups:

- **Numbering (R1 high, R2 high-caveat):** `RecEspCod = MAX+1` per unit (`psau_inc_recesp.java:56-74,124`). Legacy allocator is *read-max-then-insert with no lock* → the modern impl MUST make allocation atomic (per-unit sequence / `SELECT … FOR UPDATE` / unique+retry) to keep numbering gap-free and collision-free.
- **Auth (R3 medium):** `pisauthorized("SAU_RECESP", Gx_mode)` per-mode (`impl:2006-2033`).
- **Master validations (R4–R11 high):** date required (R4), patient required+exists (R5/R10), prescriber required+exists (R6/R11), obs ≤300 (R7), unit exists (R8), issuer optional-but-exists-if-set (R9).
- **Patient eligibility (R12 high, R13 high, R14 LOW):** block **PacSit=2 (inativo)** "Não é Possível Incluir Paciente Inativo!" (R12); **CNS-missing = non-blocking warning** (R13); **NO deceased-patient/PacObi block exists in this transaction** (R14 — potential compliance gap, see OQ3).
- **Derivations/defaults (R15/R16/R17/R19/R22/R25/R26/R30 high/medium):** patient age (R15), social-name display (R16), server-stamped `RecEspUsuLogin` (R17), line text ← `RemNom` (R19), line obs ← posology (R22), `RecInd` defaults false (R25), `RecEspSeqUlt` line counter (R26), `RecEspCon`/`RecTip` pass-through, **no state machine** (R30).
- **Child line items (R18–R24 high):** medication optional/exists-if-set (R18), **prescription text required** (R20), posology optional/exists-if-set (R21), **type required** (R23), **quantity NOT validated** — port as optional, ceilings are a regulatory OQ (R24).
- **Interaction/MPP (R27 medium):** per-unit `UniRecInterMedMpp` flag → advisory drug-interaction/MPP **warning, not a block**.
- **Audit (R28 high):** every INS/UPD/DLT writes a log row (→ `common/audit`), key = `<uni>/<cod>`, actor+timestamp, never drug/patient detail.
- **Delete (R29 high):** legacy **HARD-deletes** master + all children, no retention guard, no cancel column → **almost certainly non-compliant; do NOT silently port** (OQ2).
- **Copy (R31 medium):** clone allocates new `numero`, copies master+items preserving seqs; note legacy preserves original login/date (modern copy should re-stamp — verify).
- **Print (R32–R36):** type-3 → "RECEITUÁRIO CONTROLE ESPECIAL" (R32); **1ª via Farmácia / 2ª via Paciente** (R33); council/CRF stamped-if-present but **not validated before emit** (R34); **no print auth gate, no reprint limit/audit** (R35); denied lines not filtered from print (R36, low).

## open_questions — code-unresolvable (KB/IDE)
- **Q1** `RecTip` (master type, all 0 live) — does it encode A/B1/B2/C notification categories? Confirm in KB before exposing.
- **Q2** `RecEspCon` (status/confirm) — no lifecycle in code; is confirmation/cancellation handled elsewhere or dead?
- **Q3 (action, not regulatory)** `/migrate-slice` must re-introspect **SAU_RECESP1** and extend V14 with its 9 missing columns (see `persistence.child_v14`).

## Regulatory open_questions — HUMAN / CRF / SECURITY SIGN-OFF REQUIRED (do NOT resolve in code; block cutover)
1. **Numbering gaps & uniqueness** — legacy allocator races (R2) and hard-delete (R29) can both collide and leave gaps. Confirm Portaria 344/98 numbering rules before choosing the modern allocator.
2. **Retention / deletion** — R29 hard-deletes with no retention guard and no cancel/status column. Confirm mandated retention period; decide block-delete vs. soft/audited cancellation (may need a *deliberate* future schema milestone to add a status column — not Phase 1).
3. **Deceased-patient block** — R14: no `PacObi`/óbito check here. Confirm whether prescribing for a deceased patient must be blocked, and where (this slice also resolves SAU_PAC OQ2).
4. **Prescriber council/CRF** — R6/R11 require prescriber existence but R34 only *prints* the council if present, never validates. Confirm whether a valid active CRF/council must gate create/print.
5. **Reprint control** — R35: unlimited, unaudited reprints. Confirm reprint limits and per-emission audit.
6. **Quantity ceilings** — R24: no quantity/period/continuous-use limits. Confirm Portaria 344/98 ceilings (e.g. 30/60-day, max units) to add as new rules.
7. **SNGPC / ANVISA notification** — none found. Confirm whether the modern system must emit SNGPC notifications and under which category.
8. **Denied (indeferido) items on print** — R36: denied lines currently print; confirm intended behavior.
