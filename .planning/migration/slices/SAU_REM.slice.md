```yaml
slice: SAU_REM
domain: medicamento
description: Medicamento
wave: 3
complexity: L           # sau_rem_impl.java 708 KB / 13 998 lines
primary_table: SAU_REM
sub_tables: [SAU_REM1, SAU_REM2, SAU_REM_UNISETOR, SAU_REMPOSO]

constellation:
  transaction:  sau_rem.java                          # servlet stub; getServletInfo()="Medicamento"
  impl:         sau_rem_impl.java                     # 708 KB, 13 998 lines — rule source
  bc:           null                                  # no BC; no gxmetadata/sau_rem.json
  workwith:     hwwsau_rem.java                       # list/search screen "Consultar Medicamentos"
  workwith_impl: hwwsau_rem_impl.java                 # ~196 KB
  view:         hviewsau_rem.java                     # read-only detail
  view_impl:    hviewsau_rem_impl.java
  prompts:
    - hpromptsau_rem.java                             # "Selecionar Medicamento" — general lookup
    - hpromptsau_rem_receita.java                     # prescription-context lookup
    - hpromptsau_remtipo.java                         # tipo-de-medicamento combo lookup
    - hpromptsau_remobs.java                          # posologia combo lookup
  sections:
    - hsau_remgeneral.java                            # "Informações do Medicamento" WebComponent
    - hsau_remsau_unisetorwc.java                     # "Unidades de Atendimento" WebComponent
  related_bc:   sau_remlot_bc.java                   # 148 KB — lote BC (separate slice)
  related_sdt:  SdtSAU_REMLOT.java                   # 134 KB
  sub_trn:      sau_remobs.java                       # posologia sub-transaction (Wave 1)

depends_on:
  - SAU_TIPREM        # Wave 1 — TipRemCod FK (tipo de medicamento); SPECCED ✓
  - SAU_REMOBS        # Wave 1 — PosoRemObsCod FK via SAU_REMPOSO junction; SPECCED ✓
  - SAU_UNI           # Wave 2 — UniCod FK for SAU_REM1 + SAU_REM_UNISETOR; NOT YET SPECCED ⚠
  - SAU_UNISETOR      # Wave 2 — SetorCod FK for SAU_REM_UNISETOR; SPECCED ✓
  - SAU_APRREM        # Wave 3 cycle partner — AprRemCod FK; NOT YET SPECCED ⚠

# NOTE: SAU_DCB, SAU_RENAME, SAU_RENAMEATUALIZADO, SAU_RENAME_DEPARA, OBM,
# InteracaoMedicamentosa are referenced as read-only lookups. Migration wave TBD.
# Until migrated these FKs remain as raw-id columns in the entity.

# ── Primary table SAU_REM (28 columns — confirmed from INSERT at sau_rem_impl.java:11946)
fields:
  # ─── SAU_REM (primary)
  - { gx: RemCod,                  name: id,                                col: RemCod,                  type: integer,      pk: true,  nullable: false, note: "PK, app-assigned via psau_inc_med (MAX+1); recommend DB SEQUENCE — OQ-6" }
  - { gx: RemNom,                  name: nome,                              col: RemNom,                  type: varchar(250), pk: false, nullable: true,  picture: "@!" }
  - { gx: TipRemCod,               name: tipoMedicamentoCodigo,             col: TipRemCod,               type: integer,      pk: false, nullable: true,  note: "FK → SAU_TIPREM; 0 = not set (optional)" }
  - { gx: DcbCod,                  name: dcbCodigo,                         col: DcbCod,                  type: char(10),     pk: false, nullable: true,  note: "FK → SAU_DCB (not migrated); raw-id" }
  - { gx: RENAMECod,               name: renameCodigo,                      col: RENAMECod,               type: varchar(20),  pk: false, nullable: true,  note: "FK → SAU_RENAME (not migrated); column name ALL-CAPS" }
  - { gx: RenameAtualCod,          name: renameAtualCodigo,                 col: RenameAtualCod,          type: varchar(20),  pk: false, nullable: true,  note: "FK → SAU_RENAMEATUALIZADO (not migrated)" }
  - { gx: AprRemCod,               name: apresentacaoCodigo,                col: AprRemCod,               type: integer,      pk: false, nullable: true,  note: "FK → SAU_APRREM (Wave 3 cycle partner; raw-id until migrated)" }
  - { gx: ObmCod,                  name: obmCodigo,                         col: ObmCod,                  type: varchar(30),  pk: false, nullable: true,  note: "FK → OBM (not migrated)" }
  - { gx: RemTipoProduto,          name: tipoProduto,                       col: RemTipoProduto,          type: smallint,     pk: false, nullable: true,  note: "Enum: 0=Nenhum 1=Básico 2=Estratégico 3=Próprio 4=Especializado" }
  - { gx: RemCon,                  name: concentracao,                      col: RemCon,                  type: varchar(150), pk: false, nullable: true }
  - { gx: RemFarBas,               name: farmaciaBasica,                    col: RemFarBas,               type: smallint,     pk: false, nullable: true,  note: "boolean flag 0/1" }
  - { gx: RemPsico,                name: psicotropico,                      col: RemPsico,                type: smallint,     pk: false, nullable: true,  note: "boolean flag; Portaria 344/98 significance — regulatory" }
  - { gx: RemConEsp,               name: controleEspecial,                  col: RemConEsp,               type: smallint,     pk: false, nullable: true,  note: "boolean flag; drives SAU_RECESP — regulatory" }
  - { gx: RemEti,                  name: etico,                             col: RemEti,                  type: smallint,     pk: false, nullable: true,  note: "boolean flag" }
  - { gx: RemVlrHos,               name: valorHospitalar,                   col: RemVlrHos,               type: numeric(11,4),pk: false, nullable: true }
  - { gx: RemVlrUni,               name: valorUnitario,                     col: RemVlrUni,               type: numeric(11,4),pk: false, nullable: true }
  - { gx: RemSemRename,            name: semRename,                         col: RemSemRename,            type: smallint,     pk: false, nullable: true,  note: "boolean; skips all RENAME FK + type checks when true" }
  - { gx: RemPortariaPsicotropico, name: portariaPsicotropico,              col: RemPortariaPsicotropico, type: varchar(20),  pk: false, nullable: true,  note: "Portaria SVS/MS number for psychotropic" }
  - { gx: RemSit,                  name: situacao,                          col: RemSit,                  type: smallint,     pk: false, nullable: true,  note: "Enum: 1=ATIVO 2=INATIVO" }
  - { gx: RemOmitirSaldo,          name: omitirSaldo,                       col: RemOmitirSaldo,          type: smallint,     pk: false, nullable: true,  note: "boolean flag" }
  - { gx: RemUsarPosologia,        name: usarPosologia,                     col: RemUsarPosologia,        type: smallint,     pk: false, nullable: true,  note: "boolean; controls SAU_REMPOSO sub-panel visibility + processing" }
  - { gx: RemMPP,                  name: medicamentoPotencialmentePerigoso, col: RemMPP,                  type: smallint,     pk: false, nullable: true,  note: "boolean; MPP flag" }
  - { gx: RemMPPDes,               name: mppEfeitos,                        col: RemMPPDes,               type: varchar(1000),pk: false, nullable: true }
  - { gx: RemMPPCanMotivo,         name: mppCancelamentoMotivo,             col: RemMPPCanMotivo,         type: varchar(300), pk: false, nullable: true }
  - { gx: RemMPPCanData,           name: mppCancelamentoData,               col: RemMPPCanData,           type: timestamp,    pk: false, nullable: true }
  - { gx: RemMppCanUsuLogin,       name: mppCancelamentoLogin,              col: RemMppCanUsuLogin,       type: char(20),     pk: false, nullable: true,  note: "Careful: lowercase 'pp' in column name (case-sensitive)" }
  - { gx: RemUniSetorSeqUlt,       name: ultimaSequenciaUnidadeSetor,       col: RemUniSetorSeqUlt,       type: integer,      pk: false, nullable: true,  note: "Counter for SAU_REM_UNISETOR sequence; see R14, OQ-7" }
  - { gx: RemUsuLogin,             name: usuarioLogin,                      col: RemUsuLogin,             type: char(20),     pk: false, nullable: true,  note: "Stamped from session context on every write (R16)" }

  # ─── SAU_REM1 (Unidade do Medicamento) — composite PK (RemCod, RemUniCod)
  - { gx: RemCod,      name: id.remCod,       col: RemCod,    type: integer,  pk: true,  nullable: false, table: SAU_REM1, note: "FK → SAU_REM" }
  - { gx: RemUniCod,   name: id.remUniCod,    col: RemUniCod, type: integer,  pk: true,  nullable: false, table: SAU_REM1, note: "FK → SAU_UNI(UniCod); immutable after insert (R46)" }
  - { gx: RemEstMin,   name: estoqueMinimo,   col: RemEstMin, type: integer,  pk: false, nullable: true,  table: SAU_REM1 }
  - { gx: RemUniSit,   name: situacao,        col: RemUniSit, type: smallint, pk: false, nullable: true,  table: SAU_REM1, note: "Enum: 1=ATIVO 2=INATIVO" }

  # ─── SAU_REM2 (Código de Barras / EAN-13) — composite PK (RemCod, RemEan13)
  - { gx: RemCod,   name: id.remCod, col: RemCod,  type: integer, pk: true, nullable: false, table: SAU_REM2, note: "FK → SAU_REM" }
  - { gx: RemEan13, name: id.ean13,  col: RemEan13,type: bigint,  pk: true, nullable: false, table: SAU_REM2, note: "N(13,0) → bigint (NOT int); confirmed from Conversion XML" }

  # ─── SAU_REM_UNISETOR (Medicamento por Unidade+Setor) — composite PK (RemCod, RemUniSetorSeq, RemUniSetorUniCod)
  - { gx: RemCod,               name: id.remCod,          col: RemCod,               type: integer,  pk: true,  nullable: false, table: SAU_REM_UNISETOR, note: "FK → SAU_REM" }
  - { gx: RemUniSetorSeq,       name: id.sequencia,       col: RemUniSetorSeq,       type: integer,  pk: true,  nullable: false, table: SAU_REM_UNISETOR, note: "Assigned from RemUniSetorSeqUlt (R14)" }
  - { gx: RemUniSetorUniCod,    name: id.unidadeCodigo,   col: RemUniSetorUniCod,    type: integer,  pk: true,  nullable: false, table: SAU_REM_UNISETOR, note: "FK → SAU_UNI(UniCod); part of PK AND FK to UNISETOR" }
  - { gx: RemUniSetorSetorCod,  name: setorCodigo,        col: RemUniSetorSetorCod,  type: integer,  pk: false, nullable: true,  table: SAU_REM_UNISETOR, note: "FK → SAU_UNISETOR(UniCod,SetorCod) composite" }
  - { gx: RemUniSetorEstMin,    name: estoqueMinimo,      col: RemUniSetorEstMin,    type: integer,  pk: false, nullable: true,  table: SAU_REM_UNISETOR }
  - { gx: RemUniSetorSit,       name: situacao,           col: RemUniSetorSit,       type: smallint, pk: false, nullable: false, table: SAU_REM_UNISETOR, note: "Enum: 1=ATIVO 2=INATIVO" }

  # ─── SAU_REMPOSO (junction Medicamento ↔ Posologia) — composite PK (RemCod, PosoRemObsCod)
  - { gx: RemCod,       name: id.remCod,         col: RemCod,       type: integer, pk: true, nullable: false, table: SAU_REMPOSO, note: "FK → SAU_REM" }
  - { gx: PosoRemObsCod,name: id.posologiaCodigo, col: PosoRemObsCod,type: integer, pk: true, nullable: false, table: SAU_REMPOSO, note: "FK → SAU_REMOBS(RemObsCod)" }

  # ─── Display-only (not persisted; sourced from FK lookups in legacy UI)
  - { gx: TipRemDes,                name: tipoMedicamentoDescricao,    readOnly: true, source: SAU_TIPREM }
  - { gx: AprRemDes,                name: apresentacaoDescricao,       readOnly: true, source: SAU_APRREM }
  - { gx: DcbDes,                   name: dcbDescricao,                readOnly: true, source: SAU_DCB }
  - { gx: ObmNome,                  name: obmNome,                     readOnly: true, source: OBM }
  - { gx: RENAMEPrincAt,            name: renamePrincipioAtivo,        readOnly: true, source: SAU_RENAME }
  - { gx: RENAMEConc,               name: renameConcentracao,          readOnly: true, source: SAU_RENAME }
  - { gx: RENAMEFormFarm,           name: renameFormaFarmaceutica,     readOnly: true, source: SAU_RENAME }
  - { gx: RENAMEVol,                name: renameVolume,                readOnly: true, source: SAU_RENAME }
  - { gx: RENAMEUnd,                name: renameUnidade,               readOnly: true, source: SAU_RENAME }
  - { gx: RenameAtualDesc,          name: renameAtualDescricao,        readOnly: true, source: SAU_RENAMEATUALIZADO }
  - { gx: RenameAtualBasico,        name: renameAtualBasico,           readOnly: true, source: SAU_RENAMEATUALIZADO }
  - { gx: RenameAtualEstrategico,   name: renameAtualEstrategico,      readOnly: true, source: SAU_RENAMEATUALIZADO }
  - { gx: RenameAtualProprio,       name: renameAtualProprio,          readOnly: true, source: SAU_RENAMEATUALIZADO }
  - { gx: RenameAtualEspecializado, name: renameAtualEspecializado,    readOnly: true, source: SAU_RENAMEATUALIZADO }

keys:
  pk:
    SAU_REM:          [RemCod]
    SAU_REM1:         [RemCod, RemUniCod]
    SAU_REM2:         [RemCod, RemEan13]
    SAU_REM_UNISETOR: [RemCod, RemUniSetorSeq, RemUniSetorUniCod]
    SAU_REMPOSO:      [RemCod, PosoRemObsCod]
  indexes:
    SAU_REM:
      - { name: USAU_REM,          cols: [RemNom],                                      unique: false, note: "name search" }
      - { name: ISAU_REM1,         cols: [RENAMECod, RenameAtualCod],                   unique: false }
      - { name: ISAU_REM2,         cols: [AprRemCod],                                   unique: false }
      - { name: ISAU_REM3,         cols: [TipRemCod],                                   unique: false }
      - { name: ISAU_REM4,         cols: [ObmCod],                                      unique: false }
      - { name: ISAU_REM6,         cols: [DcbCod],                                      unique: false }
    SAU_REM1:
      - { name: ISAU_REM5,         cols: [RemUniCod],                                   unique: false }
    SAU_REM_UNISETOR:
      - { name: USAU_REM_UNISETOR, cols: [RemUniSetorUniCod, RemUniSetorSetorCod, RemCod], unique: false, note: "alternate-key; used by uniqueness check R30" }
    SAU_REMPOSO:
      - { name: ISAU_REMPOSO1,     cols: [PosoRemObsCod],                               unique: false }
  fks:
    - { from: SAU_REM,          cols: [TipRemCod],                                    ref: SAU_TIPREM,           refCols: [TipRemCod],                         note: "add FK now; SAU_TIPREM migrated" }
    - { from: SAU_REM,          cols: [DcbCod],                                       ref: SAU_DCB,              refCols: [DcbCod],              deferred: true, note: "SAU_DCB not yet migrated" }
    - { from: SAU_REM,          cols: [RENAMECod],                                    ref: SAU_RENAME,           refCols: [RENAMECod],           deferred: true }
    - { from: SAU_REM,          cols: [RenameAtualCod],                               ref: SAU_RENAMEATUALIZADO, refCols: [RenameAtualCod],      deferred: true }
    - { from: SAU_REM,          cols: [RENAMECod, RenameAtualCod],                    ref: SAU_RENAME_DEPARA,    refCols: [RENAMECod, RenameAtualCod], deferred: true, note: "composite FK via crosswalk table" }
    - { from: SAU_REM,          cols: [AprRemCod],                                    ref: SAU_APRREM,           refCols: [AprRemCod],           deferred: true, note: "Wave 3 cycle partner" }
    - { from: SAU_REM,          cols: [ObmCod],                                       ref: OBM,                  refCols: [ObmCod],              deferred: true }
    - { from: SAU_REM1,         cols: [RemCod],                                       ref: SAU_REM,              refCols: [RemCod] }
    - { from: SAU_REM1,         cols: [RemUniCod],                                    ref: SAU_UNI,              refCols: [UniCod],              note: "SAU_UNI Wave 2; specced? check before adding FK" }
    - { from: SAU_REM2,         cols: [RemCod],                                       ref: SAU_REM,              refCols: [RemCod] }
    - { from: SAU_REM_UNISETOR, cols: [RemCod],                                       ref: SAU_REM,              refCols: [RemCod] }
    - { from: SAU_REM_UNISETOR, cols: [RemUniSetorUniCod],                            ref: SAU_UNI,              refCols: [UniCod] }
    - { from: SAU_REM_UNISETOR, cols: [RemUniSetorUniCod, RemUniSetorSetorCod],       ref: SAU_UNISETOR,         refCols: [UniCod, SetorCod],    note: "composite FK; SetorCod nullable" }
    - { from: SAU_REMPOSO,      cols: [RemCod],                                       ref: SAU_REM,              refCols: [RemCod] }
    - { from: SAU_REMPOSO,      cols: [PosoRemObsCod],                                ref: SAU_REMOBS,           refCols: [RemObsCod],           note: "SAU_REMOBS Wave 1; migrated" }
  delete_guards:
    - "SAU_RECESP1 references RemCod (R21) — block delete; ON DELETE RESTRICT"
    - "SAU_REMLOT references RemCod (R22) — block delete; ON DELETE RESTRICT"
    - "InteracaoMedicamentosa references RemCod as REM1 (R20) and REM2 (R19) — block delete"
    - "SAU_REMLOT references (RemCod, UniCod) via SAU_REM1 before deleting a SAU_REM1 row (R27)"

endpoints:
  - { method: GET,    path: /api/medicamentos,                  from: hwwsau_rem,                desc: "list + search; filters: id, nome (LIKE), tipoMedicamentoCodigo, situacao, psicotropico, controleEspecial, etico; sort by id or nome" }
  - { method: GET,    path: /api/medicamentos/{id},             from: hviewsau_rem,              desc: "full detail; includes all sub-level collections and FK display fields" }
  - { method: POST,   path: /api/medicamentos,                  from: "sau_rem INS",             desc: "create with embedded sub-levels (REM1, REM2, REM_UNISETOR, REMPOSO)" }
  - { method: PUT,    path: /api/medicamentos/{id},             from: "sau_rem UPD",             desc: "update; sub-levels replaced as orphan-removable collections" }
  - { method: DELETE, path: /api/medicamentos/{id},             from: "sau_rem DLT",             desc: "delete; cascades sub-levels; blocked by R19-R22 guards" }
  - { method: GET,    path: /api/medicamentos/lookup,           from: hpromptsau_rem,            desc: "autocomplete (RemCod + RemNom); general use" }
  - { method: GET,    path: /api/medicamentos/lookup/receita,   from: hpromptsau_rem_receita,    desc: "autocomplete in prescription context; compare filter predicates — OQ-8" }
  # Supporting lookups needed by the form (served by already-migrated slices):
  - { method: GET,    path: /api/tipos-medicamento/lookup,      from: hpromptsau_remtipo,        desc: "tipo-de-medicamento combo (SAU_TIPREM)" }
  - { method: GET,    path: /api/posologias/lookup,             from: hpromptsau_remobs,         desc: "posologia combo (SAU_REMOBS)" }

rules:
  # ── Required fields ──────────────────────────────────────────────────────
  - id: R1
    when: insert|update
    desc: "RemNom (nome) must not be blank; error 'Informe o Nome do Medicamento!'"
    source: "sau_rem_impl.java:3396-3401"
    confidence: high
    test: MedicamentoServiceTest#rejectsBlankNome

  # ── FK existence checks ───────────────────────────────────────────────────
  - id: R2
    when: insert|update
    desc: "TipRemCod (tipo de medicamento) must exist in SAU_TIPREM when non-zero (0 = not set); error 'Não existe Tipos de Medicamento'"
    source: "sau_rem_impl.java:3404-3418"
    confidence: high
    test: MedicamentoServiceTest#rejectsNonExistentTipRemCod

  - id: R3
    when: insert|update
    desc: "DcbCod must exist in SAU_DCB when non-empty; error 'Não existe DCB'"
    source: "sau_rem_impl.java:3420-3434"
    confidence: high
    test: MedicamentoServiceTest#rejectsNonExistentDcbCod

  - id: R4
    when: insert|update
    desc: "AprRemCod (apresentação) must exist in SAU_APRREM when non-zero; error 'Não existe Forma de Apresentação'"
    source: "sau_rem_impl.java:3436-3450"
    confidence: high
    test: MedicamentoServiceTest#rejectsNonExistentAprRemCod

  - id: R5
    when: insert|update
    desc: "RenameAtualCod must exist in SAU_RENAMEATUALIZADO when non-empty; error 'Não existe rename atualizado com as tabelas do webservice do Hórus'. Only checked when RemSemRename is false (see R11)."
    source: "sau_rem_impl.java:3452-3478"
    confidence: high
    test: MedicamentoServiceTest#rejectsNonExistentRenameAtualCod

  - id: R6
    when: insert|update
    desc: "RENAMECod must exist in SAU_RENAME when non-empty; error 'Não existe SAU_ RENAME'"
    source: "sau_rem_impl.java:3638-3647"
    confidence: high
    test: MedicamentoServiceTest#rejectsNonExistentRENAMECod

  - id: R7
    when: insert|update
    desc: "When both RENAMECod and RenameAtualCod are provided, the pair must exist in the RENAME de-para crosswalk table; error 'Não existe Tabela de-para RENAME'. Suppressed if either is empty."
    source: "sau_rem_impl.java:3555-3567"
    confidence: high
    test: MedicamentoServiceTest#rejectsInvalidRenameDepara

  - id: R8
    when: insert|update
    desc: "ObmCod must exist in OBM when non-empty; error 'Não existe OBM'"
    source: "sau_rem_impl.java:3622-3637"
    confidence: high
    test: MedicamentoServiceTest#rejectsNonExistentObmCod

  # ── RENAME type-compatibility ─────────────────────────────────────────────
  - id: R9
    when: insert|update
    desc: >
      RemTipoProduto (0-4) must be compatible with the product-type flags of the selected
      RenameAtual record:
        1 (Básico) requires RenameAtualBasico=true;
        2 (Estratégico) requires RenameAtualEstrategico=true;
        3 (Próprio) requires RenameAtualProprio=true;
        4 (Especializado) requires RenameAtualEspecializado=true;
        0 (Nenhum) is rejected if any flag is true.
      Error: 'Tipo não é compativel, <accepted-types>'. Only checked when RemSemRename=false and RenameAtualCod is supplied.
    source: "sau_rem_impl.java:3541-3588"
    confidence: high
    test: MedicamentoServiceTest#rejectsIncompatibleTipoProduto

  - id: R10
    when: insert|update|load
    desc: "RemTipoProduto is auto-derived from the flags of the selected RenameAtual: only Básico → 1; only Estratégico → 2; only Especializado → 4; none → 0. Applied on initial load and whenever RenameAtualCod changes. Not triggered when RemSemRename=true."
    source: "sau_rem_impl.java:3479-3540, 3269-3337, 9768-9799"
    confidence: high
    test: MedicamentoServiceTest#derivesTipoProdutoFromRenameAtualFlags

  - id: R11
    when: insert|update|load
    desc: "When RemSemRename=true, RenameAtualCod is logically disabled and all RENAME FK + type-compatibility validations (R5, R7, R9, R10) are bypassed."
    source: "sau_rem_impl.java:3589-3608, 3338-3357"
    confidence: high
    test: MedicamentoServiceTest#bypassesRenameValidationWhenSemRename

  - id: R12
    when: insert|update|load
    desc: "Display-only RENAMEDescr is derived as: RENAMEPrincAt + ', ' + RENAMEConc + ', ' + RENAMEFormFarm + ', ' + RENAMEVol + ', ' + RENAMEUnd (from SAU_RENAME lookup). Returned in the response DTO."
    source: "sau_rem_impl.java:3659-3660, 3371-3372"
    confidence: high
    test: MedicamentoServiceTest#derivesRENAMEDescr

  # ── Derived counters / sequences ──────────────────────────────────────────
  - id: R13
    when: insert|update|load
    desc: "RemPosoCont (posology row count) is a denormalized counter: incremented on SAU_REMPOSO insert, decremented on delete, unchanged on update. Service must maintain it within the same transaction."
    source: "sau_rem_impl.java:3383-3394, 6949-6972"
    confidence: high
    test: MedicamentoServiceTest#derivesRemPosoCont
    notes: "Stored on SAU_REM — confirm column exists in DB; not in the 28-column INSERT list above. May be a display-only variable. Verify in DB schema."

  - id: R14
    when: insert|delete
    desc: "RemUniSetorSeqUlt is incremented by 1 on each new SAU_REM_UNISETOR INSERT (new row's RemUniSetorSeq = incremented value) and decremented on DELETE."
    source: "sau_rem_impl.java:5477-5486, 2734-2736, 2763-2765, 2785-2787"
    confidence: high
    test: MedicamentoServiceTest#maintainsRemUniSetorSeqUlt

  # ── PK generation ─────────────────────────────────────────────────────────
  - id: R15
    when: insert
    desc: "RemCod is generated by procedure psau_inc_med (MAX(RemCod)+1 from SAU_REM, or 1 if empty). Spring service should use a DB SEQUENCE instead; see OQ-6."
    source: "sau_rem_impl.java:5341-5350; psau_inc_med.java:50-73"
    confidence: high
    test: MedicamentoServiceTest#generatesRemCodOnInsert

  # ── Audit stamping ────────────────────────────────────────────────────────
  - id: R16
    when: insert|update
    desc: "RemUsuLogin is stamped with the authenticated user's login (SdtContext.UsuLogin) immediately before INSERT/UPDATE after all validations pass."
    source: "sau_rem_impl.java:5351-5353"
    confidence: high
    test: MedicamentoServiceTest#stampsRemUsuLoginFromSession

  - id: R17
    when: always
    desc: "Authorization check via pisauthorized('sau_rem', mode) on Start. Returns 0 → redirect hnotauthorized. Missing session → redirect hnotauthorizeduser. Spring service: JWT filter + @PreAuthorize per endpoint."
    source: "sau_rem_impl.java:2848-2873"
    confidence: high
    test: MedicamentoServiceTest#rejectsUnauthorizedAccess

  - id: R18
    when: always
    desc: "On the log event, psau_inc_log inserts an audit row into SAU_LOG (employer, timestamp, user, mode, program='SAU_REM', key). Spring service: replicate via @Auditable / common audit writer on every POST/PUT/DELETE."
    source: "sau_rem_impl.java:9024-9046; psau_inc_log.java:89-103"
    confidence: high
    test: MedicamentoServiceTest#writesAuditLogOnChange

  # ── Delete guards (parent) ────────────────────────────────────────────────
  - id: R19
    when: delete
    desc: "Block DELETE if InteracaoMedicamentosa references RemCod as the second drug (REM2 side); error 'Interacao Medicamentosa (Intera Medicamentosa_REM2)'"
    source: "sau_rem_impl.java:4843-4848"
    confidence: high
    test: MedicamentoServiceTest#blocksDeleteWhenReferencedByInteracaoREM2

  - id: R20
    when: delete
    desc: "Block DELETE if InteracaoMedicamentosa references RemCod as the first drug (REM1 side); error 'Interacao Medicamentosa (Intera Medicamentosa_REM1)'"
    source: "sau_rem_impl.java:4851-4857"
    confidence: high
    test: MedicamentoServiceTest#blocksDeleteWhenReferencedByInteracaoREM1

  - id: R21
    when: delete
    desc: "Block DELETE if SAU_RECESP1 (Itens do Receituário Controle Especial) references this medication; error 'Itens do Receituário Controle Especial'"
    source: "sau_rem_impl.java:4859-4865"
    confidence: high
    test: MedicamentoServiceTest#blocksDeleteWhenReferencedByRecespItens
    notes: "Regulatory guard — controlled-substance prescription items. Verify with regulatory lead before cutover."

  - id: R22
    when: delete
    desc: "Block DELETE if SAU_REMLOT (Lote de Medicamento) references this medication; error 'Lote de Medicamento'"
    source: "sau_rem_impl.java:4867-4873"
    confidence: high
    test: MedicamentoServiceTest#blocksDeleteWhenReferencedByRemlot

  # ── Cascade deletes (sub-levels) ──────────────────────────────────────────
  - id: R23
    when: delete
    desc: "On parent DELETE, cascade-delete all SAU_REMPOSO rows for this RemCod first."
    source: "sau_rem_impl.java:4618-4628"
    confidence: high
    test: MedicamentoServiceTest#cascadeDeletesPosologiaRows

  - id: R24
    when: delete
    desc: "On parent DELETE, cascade-delete all SAU_REM2 (EAN-13) rows for this RemCod first."
    source: "sau_rem_impl.java:4629-4635"
    confidence: high
    test: MedicamentoServiceTest#cascadeDeletesEan13Rows

  - id: R25
    when: delete
    desc: "On parent DELETE, cascade-delete all SAU_REM_UNISETOR rows for this RemCod first."
    source: "sau_rem_impl.java:4640-4649"
    confidence: high
    test: MedicamentoServiceTest#cascadeDeletesUniSetorRows

  - id: R26
    when: delete
    desc: "On parent DELETE, cascade-delete all SAU_REM1 rows for this RemCod first."
    source: "sau_rem_impl.java:4651-4658"
    confidence: high
    test: MedicamentoServiceTest#cascadeDeletesUniRows

  # ── SAU_REM_UNISETOR sub-level rules ──────────────────────────────────────
  - id: R27
    when: delete
    desc: "Block DELETE of a SAU_REM1 (unidade) row if SAU_REMLOT references (RemCod, UniCod); error 'Lote de Medicamento'"
    source: "sau_rem_impl.java:6388-6395"
    confidence: high
    test: MedicamentoServiceTest#blocksDeleteOfRem1WhenReferencedByRemlot

  - id: R28
    when: insert|update
    desc: "SAU_REM_UNISETOR: RemUniSetorUniCod must exist in SAU_UNI; error 'Não existe Vinculo de Medicamento com Unidade e Setor'"
    source: "sau_rem_impl.java:5537-5548"
    confidence: high
    test: MedicamentoServiceTest#rejectsNonExistentRemUniSetorUniCod

  - id: R29
    when: insert|update
    desc: "SAU_REM_UNISETOR: (RemUniSetorUniCod, RemUniSetorSetorCod) pair must exist in SAU_UNISETOR when both non-zero; error 'Não existe Vinculo de Medicamento com Unidade e Setor'"
    source: "sau_rem_impl.java:5551-5564"
    confidence: high
    test: MedicamentoServiceTest#rejectsNonExistentUniSetorPair

  - id: R30
    when: insert|update
    desc: "SAU_REM_UNISETOR: (RemUniSetorUniCod, RemUniSetorSetorCod, RemCod) must be unique across existing rows (excluding self on update); error GXM_1004 'Cód.Uni.,Cód.Setor,Código'"
    source: "sau_rem_impl.java:5566-5576"
    confidence: high
    test: MedicamentoServiceTest#rejectsDuplicateUniSetorRemCombination

  - id: R31
    when: update
    desc: "SAU_REM_UNISETOR: PK fields RemUniSetorSeq and RemUniSetorUniCod are read-only after insert; only EstMin and Sit can be changed."
    source: "sau_rem_impl.java:5487-5502"
    confidence: high
    test: MedicamentoServiceTest#rejectsChangeToPKFieldsOnUniSetorUpdate

  # ── SAU_REM1 sub-level rules ───────────────────────────────────────────────
  - id: R32
    when: insert|update
    desc: "SAU_REM1: RemUniCod must exist in SAU_UNI; error 'Não existe Unidade'"
    source: "sau_rem_impl.java:6077-6089"
    confidence: high
    test: MedicamentoServiceTest#rejectsNonExistentRemUniCod

  - id: R33
    when: insert
    desc: "SAU_REM2: No FK validation; only PK duplicate check — reject duplicate (RemCod, RemEan13). No EAN-13 check-digit validation in legacy code."
    source: "sau_rem_impl.java:2521-2527, 6531-6536"
    confidence: high
    test: MedicamentoServiceTest#rejectsDuplicateEan13
    notes: "Adding EAN-13 check-digit validation would be a NEW rule, not a port. Confirm with product owner — OQ-5."

  # ── SAU_REMPOSO rules ─────────────────────────────────────────────────────
  - id: R34
    when: insert|update
    desc: "SAU_REMPOSO: PosoRemObsCod must exist in SAU_REMOBS; error 'Não existe Vinculo de medicamento com a tabela de posologia'"
    source: "sau_rem_impl.java:6936-6948"
    confidence: high
    test: MedicamentoServiceTest#rejectsNonExistentPosoRemObsCod

  - id: R35
    when: insert
    desc: "SAU_REMPOSO: Duplicate (RemCod, PosoRemObsCod) PK is rejected with DuplicatePrimaryKey error."
    source: "sau_rem_impl.java:2410-2414"
    confidence: high
    test: MedicamentoServiceTest#rejectsDuplicatePosoRow

  # ── Post-save side effect ─────────────────────────────────────────────────
  - id: R36
    when: insert|update
    desc: "After a successful save, MedicamentoAtualizaUniAtendimento is called: deletes all SAU_REM1 rows for this RemCod, then re-inserts from current SAU_REM_UNISETOR (propagating UniCod + status). Synchronizes the attendance-unit table from the unit-sector configuration."
    source: "sau_rem_impl.java:2892-2895; medicamentoatualizauniatendimento.java:43-100"
    confidence: high
    test: MedicamentoServiceTest#propagatesUniAtendimentoAfterSave
    notes: "This side-effect must run within the same DB transaction. See OQ-7 for FK cascade concern."

  # ── Enum constraints ──────────────────────────────────────────────────────
  - id: R37
    when: insert|update
    desc: "RemSit must be 1 (ATIVO) or 2 (INATIVO). Validated at DTO/service level."
    source: "sau_rem_impl.java:480-489"
    confidence: high
    test: MedicamentoServiceTest#rejectsInvalidRemSit

  - id: R38
    when: insert|update
    desc: "RemTipoProduto must be 0-4 (Nenhum/Básico/Estratégico/Próprio/Especializado)."
    source: "sau_rem_impl.java:404-413"
    confidence: high
    test: MedicamentoServiceTest#rejectsInvalidRemTipoProduto

  - id: R39
    when: insert|update
    desc: "RemUniSetorSit must be 1 (ATIVO) or 2 (INATIVO)."
    source: "sau_rem_impl.java:456-463"
    confidence: high
    test: MedicamentoServiceTest#rejectsInvalidRemUniSetorSit

  - id: R40
    when: insert|update
    desc: "RemUniSit (SAU_REM1) must be 1 (ATIVO) or 2 (INATIVO)."
    source: "sau_rem_impl.java:465-474"
    confidence: high
    test: MedicamentoServiceTest#rejectsInvalidRemUniSit

  # ── Boolean flags ─────────────────────────────────────────────────────────
  - id: R41
    when: insert|update
    desc: "RemConEsp (Sujeito a Controle Especial) stored as-is (0/1). No additional service-layer validation in legacy code; drives SAU_RECESP downstream. Confirm with regulatory lead whether toggling requires special authorization — OQ-2."
    source: "sau_rem_impl.java:440-444"
    confidence: medium
    test: MedicamentoServiceTest#storesRemConEspFlag

  - id: R42
    when: insert|update
    desc: "RemPsico=1 (Psicotrópico) requires RemPortariaPsicotropico to be non-blank (regulatory requirement, Portaria SVS/MS 344/98). Defaults to 0 on insert. The legacy GeneXus code does not enforce this check; the new service must add it."
    source: "sau_rem_impl.java:435-439, 3087-3092; regulatory sign-off 2026-06-21"
    confidence: high
    test: MedicamentoServiceTest#requiresPortariaWhenPsicotropico

  - id: R43
    when: insert|update
    desc: "RemMPP (boolean) and RemMPPDes (text) are stored without additional server-side validation. Whether clearing RemMPP=false requires the cancellation fields is not enforced in sau_rem_impl — see R44 and OQ-4."
    source: "sau_rem_impl.java:4298-4408"
    confidence: medium
    test: MedicamentoServiceTest#storesMppFields

  - id: R44
    when: update
    desc: "When RemMPP transitions from true to false (MPP cancellation), RemMPPCanMotivo must be non-blank (error: 'Informe o motivo de cancelamento do MPP'). RemMppCanUsuLogin is auto-stamped from the session user login and RemMPPCanData is auto-stamped with the current timestamp. Users must not supply these fields directly — they are server-set."
    source: "sau_rem_impl.java:4298-4427; confirmed 2026-06-21"
    confidence: high
    test: MedicamentoServiceTest#requiresMppCancellationMotivoWhenClearingMppFlag

  # ── Conditional visibility / processing ───────────────────────────────────
  - id: R45
    when: insert|update|load
    desc: "SAU_REMPOSO sub-panel is only processed when RemUsarPosologia=true. When false, posologia rows in the request payload must be ignored (or rejected). Primarily a UI rule but service must not save REMPOSO rows when this flag is false."
    source: "sau_rem_impl.java:3609-3621, 3358-3370"
    confidence: high
    test: MedicamentoServiceTest#hidesRemposoPanelWhenUsarPosologiaFalse

  # ── PK immutability ────────────────────────────────────────────────────────
  - id: R46
    when: update
    desc: "SAU_REM1: RemUniCod is read-only after insert; only RemEstMin and RemUniSit can be modified."
    source: "sau_rem_impl.java:6472-6477"
    confidence: high
    test: MedicamentoServiceTest#rejectsChangeToPKFieldsOnRem1Update

  # ── Optimistic concurrency ────────────────────────────────────────────────
  - id: R47
    when: update|delete
    desc: "Optimistic lock on SAU_REM: re-reads all 28 tracked columns before UPDATE/DELETE and compares with snapshot taken at load; raises 'GXM_waschg (SAU_REM)' if any differ. Spring service: implement via @Version or ETag."
    source: "sau_rem_impl.java:4273-4469"
    confidence: high
    test: MedicamentoServiceTest#rejectsStaleUpdate

  - id: R48
    when: update|delete
    desc: "Optimistic lock on SAU_REM1: checks RemEstMin + RemUniSit unchanged since load; raises 'GXM_waschg (SAU_REM1)' if concurrent modification detected."
    source: "sau_rem_impl.java:6200-6217"
    confidence: high
    test: MedicamentoServiceTest#rejectsStaleRem1Update

  # ── Duplicate detection ───────────────────────────────────────────────────
  - id: R49
    when: insert
    desc: "SAU_REM_UNISETOR: duplicate PK (RemCod, RemUniSetorSeq, RemUniSetorUniCod) on insert returns DuplicatePrimaryKey error."
    source: "sau_rem_impl.java:2741-2745"
    confidence: high
    test: MedicamentoServiceTest#rejectsDuplicateUniSetorRow

  # ── Defaults ──────────────────────────────────────────────────────────────
  - id: R50
    when: insert
    desc: "RemPsico, RemConEsp, and RemUsarPosologia default to 0 (false) on new medication creation if not provided."
    source: "sau_rem_impl.java:3087-3103, 8568-8576"
    confidence: high
    test: MedicamentoServiceTest#defaultsBooleanFlagsOnInsert

  # ── Auth / session ────────────────────────────────────────────────────────
  - id: R51
    when: always
    desc: "Every endpoint requires authentication (GX session → JWT). Missing or invalid credentials return HTTP 401/403."
    source: "sau_rem_impl.java:355-374"
    confidence: high
    test: MedicamentoServiceTest#rejectsUnauthenticatedRequest

  - id: R52
    when: load
    desc: "On GET detail, 'medicamentounidadesvinculadas' computes a string of linked units from SAU_REM_UNISETOR for display. Verify if the Spring API needs this pre-computed summary in the response or if the client derives it from the sub-collection."
    source: "sau_rem_impl.java:2876-2886"
    confidence: high
    test: MedicamentoServiceTest#loadsLinkedUnitsOnOpen

  - id: R53
    when: update
    desc: "RemConEsp cannot be toggled from 1 to 0 (controlled-substance flag unset) while active SAU_RECESP prescriptions reference this medication. Apply the same referential check as the delete guard R21. Error: 'Não é possível remover o controle especial: existem receituários ativos para este medicamento'. Regulatory requirement — Portaria SVS/MS 344/98."
    source: "regulatory sign-off 2026-06-21"
    confidence: high
    test: MedicamentoServiceTest#blocksUnsetControleEspecialWithActivePrescriptions

phi_fields: []
# SAU_REM is a medication catalog — no patient-identifying data.
# RemUsuLogin / RemMppCanUsuLogin are system-user logins (health staff), not patient PHI.
# PHI enters only in SAU_RECESP / SAU_REMLOT which reference RemCod as FK.

auth:
  roles_required: [SAUDE_CADASTRO]
  notes: >
    Legacy enforces pisauthorized('sau_rem', mode) (sau_rem_impl.java:2855-2871).
    No hardcoded module number found; access is program-name-based.
    Confirm exact SAU_PRFCON permission code by querying live DB: SELECT * FROM SAU_PRFCON WHERE PrgNom IN ('SAU_REM','HWWSAU_REM').
    RemConEsp=1 medications (controlled substances) have no additional auth gate in sau_rem_impl;
    confirm with regulatory lead whether a separate role is needed for CRUD of controlled-substance medications.

parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios:
    - "list default (all, order by id)"
    - "list filter by nome (partial, case-insensitive)"
    - "list filter by tipoMedicamentoCodigo"
    - "list filter by situacao=ATIVO"
    - "list filter psicotropico flag"
    - "list filter controleEspecial flag"
    - "GET by id — existing medicamento with all sub-levels"
    - "GET by id — 404 for unknown RemCod"
    - "POST valid: minimal (nome + situacao)"
    - "POST valid: full payload including SAU_REM1, EAN-13, unidade-setor, posologia"
    - "POST duplicate nome (allowed — check)"
    - "POST invalid TipRemCod FK (reject)"
    - "POST RemConEsp=1 (controlled substance)"
    - "PUT update nome and situacao"
    - "PUT add new SAU_REM1 row (new unidade)"
    - "PUT remove EAN-13 row"
    - "PUT SAU_REM_UNISETOR uniqueness violation (reject)"
    - "DELETE unused medicamento (success)"
    - "DELETE with SAU_RECESP1 reference (reject)"
    - "DELETE with SAU_REMLOT reference (reject)"
    - "DELETE with InteracaoMedicamentosa reference (reject)"
    - "lookup by nome partial (hpromptsau_rem)"
    - "lookup in receita context (hpromptsau_rem_receita)"
    - "MPP flag set / unset (RemMPP + RemMPPDes)"
    - "RemConEsp=1 medication visible in receita lookup"

status: verified

# Backend + frontend + tests generated, reviewed, and parity-verified on 2026-06-21.
parity_result:
  date: 2026-06-21
  method: "DB-state comparison; new app :8090 vs shared snapshot saude-mandaguari (799 SAU_REM rows); legacy Tomcat host.docker.internal:8080"
  report: "backend/src/test/resources/parity/medicamento/PARITY-REPORT.md (+ scenarios.json)"
  summary: "23/25 PARITY"
  diverge:
    - { scenarios: [23, 25], rule: "RD6/OQ-8", detail: "GET /api/medicamentos/lookup/receita not implemented → 404 (needs legacy filter-predicate comparison)", disposition: "deferred — cutover caveat for prescription-lookup only" }
  schema_fix_found: "OQ-1 (BLOCK, fixed): RemMPP/RemOmitirSaldo/RemSemRename/RemUsarPosologia are BOOLEAN in the live DB (spec/V3 said smallint). Entity mapped Short → SQLState 22003 on every read. Fixed: entity+DTO→Boolean; V5 idempotently converts the Flyway/Testcontainers columns to BOOLEAN; frontend→boolean. Full mvn verify green; all 25 scenarios run; snapshot restored (799 rows, 0 residue)."
  notes:
    - "Verified with Flyway disabled + ddl-auto=none; seq_sau_rem_cod created manually in the snapshot (V5-equivalent)."

# Backend + frontend + tests generated and reviewed on 2026-06-21 (/migrate-slice SAU_REM).
# Domain: medicamento/ — Medicamento (SAU_REM, 28 cols) + 4 composite-PK sub-entities (@IdClass).
# Tests: MedicamentoServiceTest 20 + MedicamentoSubServiceTest 12 + MedicamentoControllerIT 8 (Testcontainers).
# Full `mvn verify` GREEN (unit 163, integ 226, 0 fail). Frontend tsc clean. Parity: stubs only (run /verify-parity).
# Schema OQs resolved against the live saude-mandaguari DB (799 medicamentos, max RemCod=17821):
schema_verified_2026_06_21:
  - "OQ-1: column types match spec; CHAR cols DcbCod/RemUsuLogin/RemMppCanUsuLogin mapped via @JdbcTypeCode(Types.CHAR)."
  - "OQ-6: no RemCod sequence in live DB → created seq_sau_rem_cod (V5, seeded MAX+1)."
  - "OQ-11: RemPosoCont is NOT a physical column → R13 implemented as a live count(*) (posologiaCount), entity omits it."

code_review:
  reviewer: migration-reviewer
  date: 2026-06-21
  fixed:
    - "R36 (BLOCK): syncUnidadesFromUnidadeSetor was destructive — dropped SAU_REM1.estoqueMinimo and ignored the R27 SAU_REMLOT guard (OQ-7 hazard). Rewritten as a non-destructive reconcile: upsert preserving estoqueMinimo, delete dropped unidades only when not remlot-referenced."
    - "R14 (BLOCK): RemUniSetorSeqUlt was decremented on delete → seq reuse / PK collision. Now an increment-only high-water mark."
    - "List filters (FLAG): GET /api/medicamentos now supports nome, tipoMedicamentoCodigo, situacao, psicotropico, controleEspecial, etico via JpaSpecificationExecutor (was nome-only)."
    - "Coverage (BLOCK): added MedicamentoSubServiceTest (12: R27/R28/R29/R30/R32/R33/R34/R35/R45 + R14 + cascade) and main FK/guard tests R3/R4/R6/R8/R20."
  divergences_accepted:
    - id: RD1
      finding: "Sub-levels use separate REST sub-resource endpoints (/unidades, /codigos-barras, /unidade-setores, /posologias) instead of an embedded POST/PUT payload (spec endpoints 157-159). R36 runs per sub-call rather than once per parent save."
      disposition: "Accepted design choice — mirrors the verified SAU_UNI pattern. Parity write scenarios re-expressed against the sub-resource API."
    - id: RD2
      finding: "@IdClass used for composite PKs (spec text suggested @EmbeddedId). Functionally equivalent; consistent with SAU_UNI."
      disposition: accepted
    - id: RD3
      finding: "Required-field violations return 422 (BusinessRule) while @Size returns 400 — same status-code split as SAU_UNI RF6."
      disposition: accepted
  deferred:
    - id: RD4
      finding: "Optimistic locking R47/R48 not ported — physical SAU_REM has no version column (no schema mutation). Concurrent edits last-writer-wins (legacy raises GXM_waschg)."
      action: "Add @Version/ETag in a later phase; needs schema sign-off."
    - id: RD5
      finding: "R18 audit writes a structured log line only; persistent SAU_LOG write is shared-infra TODO."
      action: "Implement when the sau_log slice lands."
    - id: RD6
      finding: "GET /api/medicamentos/lookup/receita (OQ-8) not implemented — needs legacy filter-predicate comparison."
      action: "Resolve OQ-8 and add the receita-context lookup."
    - id: RD7
      finding: "EAN-13 check-digit validation (OQ-5) intentionally NOT added — legacy does not validate; would be a new rule."
      action: "Confirm with product owner before adding."

open_questions:
  - id: OQ-1
    topic: schema
    resolved: true
    desc: "RESOLVED 2026-06-21 (during /verify-parity, live DB introspected): types match EXCEPT RemMPP/RemOmitirSaldo/RemSemRename/RemUsarPosologia are BOOLEAN (not smallint). Entity/DTO→Boolean; V5 converts Flyway columns. CHAR cols (DcbCod/RemUsuLogin/RemMppCanUsuLogin) via @JdbcTypeCode. See parity_result.schema_fix_found."

  - id: OQ-2
    topic: regulatory
    resolved: true
    desc: "RESOLVED 2026-06-21: Yes — RemConEsp cannot be unset when active SAU_RECESP prescriptions exist. Implemented as R53."

  - id: OQ-3
    topic: regulatory
    resolved: true
    desc: "RESOLVED 2026-06-21: Yes — RemPortariaPsicotropico is required when RemPsico=1. Implemented as R42 (upgraded to high confidence)."

  - id: OQ-4
    topic: business-rule
    resolved: true
    desc: "RESOLVED 2026-06-21: Yes — clearing RemMPP=false requires RemMPPCanMotivo non-blank; RemMppCanUsuLogin + RemMPPCanData are server-stamped automatically. Implemented as R44 (upgraded to high confidence)."

  - id: OQ-5
    topic: business-rule
    desc: "EAN-13 (SAU_REM2): should the service validate the EAN-13 check digit? Legacy code does not. Confirm with product owner — this would be a new rule, not a port."
    source: "sau_rem_impl.java:6531-6536"

  - id: OQ-6
    topic: schema
    desc: "RemCod PK generation: legacy uses psau_inc_med (MAX+1). Concurrent inserts are not safe without a lock or sequence. Verify if the live PostgreSQL schema has a SEQUENCE or auto-increment for RemCod; if not, add one in V3 migration."
    source: "psau_inc_med.java:55-72"

  - id: OQ-7
    topic: schema
    desc: "MedicamentoAtualizaUniAtendimento (R36) deletes all SAU_REM1 rows then re-inserts from SAU_REM_UNISETOR. If SAU_REMLOT references (RemCod, UniCod) of a SAU_REM1 row, the delete will fail (R27 guard applies). Verify FK cascade behavior and whether this scenario can occur in practice."
    source: "medicamentoatualizauniatendimento.java:43-50"

  - id: OQ-8
    topic: api
    desc: "hpromptsau_rem_receita: compare SQL filter predicates against hpromptsau_rem to identify any extra filters (e.g. situacao=ATIVO only, or RemConEsp filter). Required to implement the /api/medicamentos/lookup/receita endpoint correctly."

  - id: OQ-9
    topic: api
    desc: "SAU_REMOBS sub-transaction: is posologia managed exclusively within the medicamento form (via SAU_REMPOSO links) or also via a standalone /api/posologias CRUD? Confirm before designing the posologia lookup endpoint."

  - id: OQ-10
    topic: auth
    desc: "Exact SAU_PRFCON permission code for SAU_REM: query live DB SELECT * FROM SAU_PRFCON WHERE PrgNom IN ('SAU_REM', 'HWWSAU_REM') to confirm the role name before implementing @PreAuthorize."
    source: "sau_rem_impl.java:2855-2871"

  - id: OQ-11
    topic: schema
    desc: "RemPosoCont counter: the rule-miner found a posology-row-count field maintained in sau_rem_impl but it is NOT in the 28-column INSERT list. Verify in DB whether RemPosoCont is a separate column on SAU_REM or a display-only computed variable."
    source: "sau_rem_impl.java:3383-3394"

  - id: OQ-12
    topic: schema
    desc: "SAU_UNI is referenced by SAU_REM1 and SAU_REM_UNISETOR but has no slice spec yet (Wave 2). FK constraints to SAU_UNI must remain deferred until SAU_UNI is specced and migrated."

  - id: OQ-13
    topic: business-rule
    desc: "RemSemRename authorization: is there any guard on who can set RemSemRename=true? Not visible in sau_rem_impl. Verify in KB whether specific profiles are required."
    source: "sau_rem_impl.java:3589-3608"
```

---

## Domain notes

### Aggregate structure

SAU_REM is a **complex aggregate root** with five physical tables:

```
SAU_REM (Medicamento — 28 cols)
  ├── SAU_REM1[]        (MedicamentoUnidade — stock/status per health unit)
  ├── SAU_REM2[]        (MedicamentoCodigoBarras — EAN-13 barcodes)
  ├── SAU_REM_UNISETOR[] (MedicamentoUnidadeSetor — unit+sector assignment; drives SAU_REM1 via R36)
  └── SAU_REMPOSO[]     (MedicamentoPosologia — junction to SAU_REMOBS posologies)
```

**R36 side-effect:** After every save, `MedicamentoAtualizaUniAtendimento` rebuilds SAU_REM1 from SAU_REM_UNISETOR. This means SAU_REM1 is a **derived read cache** — the source of truth is SAU_REM_UNISETOR. In the Spring service, both must be updated within the same transaction.

### Regulatory significance

- `RemConEsp=1` (Sujeito a Controle Especial): flags a medication as requiring a controlled-substance prescription (`SAU_RECESP`). Portaria SVS/MS 344/98.
- `RemPsico=1` (Psicotrópico): subset of controlled substances; `RemPortariaPsicotropico` stores the applicable portaria number.
- These flags affect downstream slices (SAU_RECESP lookup filters, prescription validation). Do not change their semantics without regulatory sign-off.

### Wave-3 cycle: SAU_REM ↔ SAU_APRREM

`AprRemCod` FK references SAU_APRREM, which is migrated in the same wave. Until SAU_APRREM is specced and migrated, `AprRemCod` remains a raw integer column in the entity (no `@ManyToOne`). Add the FK constraint and JPA relationship in a follow-up migration after SAU_APRREM is merged.

### RENAME reference data (SAU_RENAME, SAU_RENAMEATUALIZADO, SAU_RENAME_DEPARA)

These tables implement the **RENAME** (Relação Nacional de Medicamentos Essenciais) federal drug registry linkage and the Hórus webservice update mechanism. Their migration wave is unassigned in BACKLOG.md. Until migrated, the three FK columns (`RENAMECod`, `RenameAtualCod`, composite de-para) remain raw-id columns. Service must still enforce validations R5–R12 by calling the legacy tables directly (via read-only JPA entities or native queries on the existing schema).

### Flyway V3 required — V1/V2 baseline stubs are wrong

| Table | V1 stub problem | Fix in V3 |
|---|---|---|
| SAU_REM | Stub has only RemCod+TipRemCod (missing 26 columns) | Drop stub, add full 28-column DDL |
| SAU_REM1 | Wrong PK (single col), RemUniCod nullable, missing RemEstMin/RemUniSit | Drop stub, recreate with correct composite PK |
| SAU_REM2 | Not present in V1 at all | Add fresh DDL |
| SAU_REM_UNISETOR | V1+V2 have wrong 2-col PK; RemUniSetorSeq is SMALLINT not integer; RemUniSetorUniCod nullable | Drop PK constraint, fix to 3-col PK, fix type, set NOT NULL |
| SAU_REMPOSO | V1 stub is correct | No changes needed |

### JPA entity summary

| Entity class | Table | PK strategy |
|---|---|---|
| `Medicamento` | SAU_REM | `@Id Integer codigo` — app-assigned |
| `MedicamentoUnidade` | SAU_REM1 | `@EmbeddedId MedicamentoUnidadeId(remCod, remUniCod)` |
| `MedicamentoCodigoBarras` | SAU_REM2 | `@EmbeddedId MedicamentoCodigoBarrasId(remCod, ean13: Long)` |
| `MedicamentoUnidadeSetor` | SAU_REM_UNISETOR | `@EmbeddedId MedicamentoUnidadeSetorId(remCod, sequencia, unidadeCodigo)` |
| `MedicamentoPosologia` | SAU_REMPOSO | `@EmbeddedId MedicamentoPosologiaId(remCod, posologiaCodigo)` |

Boolean flags (`RemFarBas`, `RemPsico`, `RemConEsp`, `RemEti`, `RemSemRename`, `RemOmitirSaldo`, `RemUsarPosologia`, `RemMPP`) stored as `Short` in entity; exposed as `Boolean` in DTO via MapStruct converters.

### Rule count by confidence

| Confidence | Count | Rules |
|---|---|---|
| high | 51 | R1-R32, R34-R36, R37-R40, R42, R44, R45-R53 |
| medium | 2 | R41, R43 |
| low | 0 | — |
| **total** | **53** | |
