```yaml
slice: SAU_UNI
domain: unidade
description: Unidade de Atendimento
wave: 2
complexity: XL    # sau_uni_impl.java 1 081 651 B / 19 945 lines
primary_table: SAU_UNI
sub_tables: [SAU_UNI1, SAU_UNI2, SAU_UNI3, SAU_UNISALA]

constellation:
  transaction:     sau_uni.java                # servlet stub; Description "Unidade de Atendimento"
  impl:            sau_uni_impl.java           # 1 081 651 B / 19 945 lines — rule source
  bc:              null                        # not a GeneXus BC; no gxmetadata/sau_uni.json
  workwith:        null                        # no hwwsau_uni.java — list UI is embedded in prompt
  view:            null                        # no hviewsau_uni.java
  prompts:
    - hpromptsau_uni.java                      # "Selecionar Unidade de Atendimento" (lookup stub)
    - hpromptsau_uni_impl.java                 # 84 431 B — lookup/autocomplete grid
  sections:        null
  sdt:             null

depends_on:
  - table: SYS_MUN
    wave: 0
    specced: false
    note: "FK SAU_UNI.UniMunCod → SYS_MUN.MunCod; SYS_MUN stub in V1 — FK added in V4"
  - table: SAU_DIS
    wave: 2
    specced: false
    note: "FK SAU_UNI.UniDisCod → SAU_DIS.DisCod; FK present in V1"
  - table: SAU_PRO
    wave: 4
    specced: false
    deferred: true
    note: "4 FKs (RespCod/DirCod/AudCod/AutCod → ProPesCod); also FK from SAU_UNI1.UniProPesCod; add in Wave-4 migration"
  - table: SAU_PROESP
    wave: 4
    specced: false
    deferred: true
    note: "SAU_UNI2 and SAU_UNI3 composite FKs to SAU_PROESP(ProPesCod,EspCod); add in Wave-4 migration"
  - table: SAU_UNISETOR
    wave: 2
    specced: true
    note: "Child table of SAU_UNI; uni↔unisetor cycle; SAU_UNI must exist before UNISETOR rows"

# ── V1/V4 baseline status ──────────────────────────────────────────────────
# SAU_UNI: FULLY DEFINED IN V1 (all 55 columns). V4 corrects 3 CHAR/VARCHAR type
# mismatches (UniCnpj, UniCep, UniLicFun) and adds 7 missing GeneXus indexes.
# SAU_UNI1, SAU_UNI2, SAU_UNI3, SAU_UNISALA: CREATED IN V4.

fields:
  # ── SAU_UNI (primary — 55 cols; authoritative: SAU_UNIConversion.xml line 1267) ────────
  - { gx: UniCod,                col: UniCod,                name: id,                                  sql_type: "integer",     pk: true,  nullable: false }
  - { gx: UniNom,                col: UniNom,                name: nome,                                sql_type: "VARCHAR(50)", pk: false, nullable: true,  note: "Stored UPPERCASE (R32)" }
  - { gx: UniRazSoc,             col: UniRazSoc,             name: razaoSocial,                         sql_type: "VARCHAR(50)", pk: false, nullable: true,  note: "Stored UPPERCASE (R34)" }
  - { gx: UniCnpj,               col: UniCnpj,               name: cnpj,                                sql_type: "CHAR(18)",    pk: false, nullable: true,  note: "V1 had VARCHAR — fixed to CHAR in V4. CNPJ validation (R12)." }
  - { gx: UniCep,                col: UniCep,                name: cep,                                 sql_type: "CHAR(8)",     pk: false, nullable: true,  note: "V1 had VARCHAR — fixed in V4. Required (R7)." }
  - { gx: UniEnd,                col: UniEnd,                name: endereco,                            sql_type: "VARCHAR(70)", pk: false, nullable: true,  note: "Required (R8); UPPERCASE (R35)" }
  - { gx: UniEndNum,             col: UniEndNum,             name: numero,                              sql_type: "VARCHAR(10)", pk: false, nullable: true,  note: "Required (R9)" }
  - { gx: UnEndCom,              col: UnEndCom,              name: complemento,                         sql_type: "VARCHAR(40)", pk: false, nullable: true,  note: "Typo in col name: UnEndCom not UniEndCom" }
  - { gx: UniBai,                col: UniBai,                name: bairro,                              sql_type: "VARCHAR(70)", pk: false, nullable: true,  note: "Free text, not FK to SAU_BAI. Required (R10). UPPERCASE (R35)." }
  - { gx: UniMunCod,             col: UniMunCod,             name: municipioId,                         sql_type: "integer",     pk: false, nullable: true,  note: "FK → SYS_MUN. Required non-zero (R11). Range 0-999999999 (R16)." }
  - { gx: UniFon,                col: UniFon,                name: telefone,                            sql_type: "VARCHAR(20)", pk: false, nullable: true,  note: "Brazilian phone regex (R13)" }
  - { gx: UniFax,                col: UniFax,                name: fax,                                 sql_type: "VARCHAR(20)", pk: false, nullable: true,  note: "Brazilian phone regex (R14)" }
  - { gx: UniLicFun,             col: UniLicFun,             name: licencaFuncionamento,                sql_type: "CHAR(10)",    pk: false, nullable: true,  note: "V1 had VARCHAR — fixed in V4" }
  - { gx: UniRes,                col: UniRes,                name: nomeResponsavel,                     sql_type: "VARCHAR(50)", pk: false, nullable: true,  note: "Natural person name. UPPERCASE (R35). @Auditable on reads." }
  - { gx: UniEMail,              col: UniEMail,              name: email,                               sql_type: "VARCHAR(70)", pk: false, nullable: true }
  - { gx: UniCnes,               col: UniCnes,               name: cnes,                                sql_type: "integer",     pk: false, nullable: true,  note: "CNES code (7-digit). Range 0-9999999 (R15). Required when program flags set (R18)." }
  - { gx: UniBPA,                col: UniBPA,                name: realizaBpa,                          sql_type: "smallint",    pk: false, nullable: true }
  - { gx: UniSIPNI,              col: UniSIPNI,              name: realizaSipni,                        sql_type: "smallint",    pk: false, nullable: true }
  - { gx: UniOrgEmi,             col: UniOrgEmi,             name: orgaoEmissor,                        sql_type: "VARCHAR(10)", pk: false, nullable: true,  note: "UPPERCASE. When non-blank, requires DirCod (R19) and AutCod (R20)." }
  - { gx: UniProPesRespCod,      col: UniProPesRespCod,      name: medicoResponsavelId,                 sql_type: "bigint",      pk: false, nullable: true,  note: "FK → SAU_PRO (deferred Wave 4). FK check in service (R27)." }
  - { gx: UniProPesDirCod,       col: UniProPesDirCod,       name: diretorId,                           sql_type: "bigint",      pk: false, nullable: true,  note: "FK → SAU_PRO (deferred). Must have CNS (R23). Required when OrgEmi set (R19)." }
  - { gx: UniProPesAudCod,       col: UniProPesAudCod,       name: profissionalAuditorId,               sql_type: "bigint",      pk: false, nullable: true,  note: "FK → SAU_PRO (deferred). Checked for existence (R28)." }
  - { gx: UniProPesAutCod,       col: UniProPesAutCod,       name: autorizadorId,                       sql_type: "bigint",      pk: false, nullable: true,  note: "FK → SAU_PRO (deferred). Must have CNS (R25). Required when OrgEmi set (R20)." }
  - { gx: UniEsfAdm,             col: UniEsfAdm,             name: esferaAdministrativa,                sql_type: "smallint",    pk: false, nullable: true,  note: "Enum: 1=PÚBLICO, 2=PRIVADO" }
  - { gx: UniPSF,                col: UniPSF,                name: realizaPsf,                          sql_type: "smallint",    pk: false, nullable: true }
  - { gx: UniSisPreNatal,        col: UniSisPreNatal,        name: realizaSisPrenatal,                  sql_type: "smallint",    pk: false, nullable: true }
  - { gx: UniHiperdia,           col: UniHiperdia,           name: realizaHiperdia,                     sql_type: "smallint",    pk: false, nullable: true }
  - { gx: UniGes,                col: UniGes,                name: gestao,                              sql_type: "smallint",    pk: false, nullable: true,  note: "Enum: 1=MUNICIPAL, 2=ESTADUAL. Default 1 on insert (R2)." }
  - { gx: UniSia,                col: UniSia,                name: sia,                                 sql_type: "VARCHAR(7)",  pk: false, nullable: true }
  - { gx: UniSigla,              col: UniSigla,              name: sigla,                               sql_type: "VARCHAR(6)",  pk: false, nullable: true,  note: "UPPERCASE (R33)" }
  - { gx: UniSit,                col: UniSit,                name: situacao,                            sql_type: "smallint",    pk: false, nullable: true,  note: "Enum: 1=ATIVADO, 2=DESATIVADO. Default 1 on insert (R3)." }
  - { gx: UniDisCod,             col: UniDisCod,             name: distritoId,                          sql_type: "smallint",    pk: false, nullable: true,  note: "FK → SAU_DIS. FK check in service (R29)." }
  - { gx: UniSIASUS,             col: UniSIASUS,             name: siaSus,                              sql_type: "VARCHAR(7)",  pk: false, nullable: true }
  - { gx: UniScnesID,            col: UniScnesID,            name: scnesId,                             sql_type: "VARCHAR(20)", pk: false, nullable: true }
  - { gx: UniExpEsus,            col: UniExpEsus,            name: exportaEsus,                         sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniExpBNAFAR,          col: UniExpBNAFAR,          name: exportaBnafar,                       sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniCadCNS,             col: UniCadCNS,             name: permitirCadastroPacienteSemCns,      sql_type: "BOOLEAN",     pk: false, nullable: true,  note: "Governs patient-registration CNS rule — critical policy flag" }
  - { gx: UniCadEnd,             col: UniCadEnd,             name: permitirCadastroPacienteSemEndereco, sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniAteSemCNS,          col: UniAteSemCNS,          name: permitirAtendimentoSemCns,           sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniAteSemEnd,          col: UniAteSemEnd,          name: permitirAtendimentoSemEndereco,      sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniEncFisio,           col: UniEncFisio,           name: permitirEncaminhamentoFisioterapia,  sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniExt,                col: UniExt,                name: fornecedorExterno,                   sql_type: "BOOLEAN",     pk: false, nullable: true,  note: "Default false (R4). Controls section visibility (R83)." }
  - { gx: UniForPesCod,          col: UniForPesCod,          name: fornecedorPessoaId,                  sql_type: "bigint",      pk: false, nullable: true,  note: "Range 0-99999999999999 (R17). FK target TBD — OQ4." }
  - { gx: TipUniCod,             col: TipUniCod,             name: tipoUnidadeId,                       sql_type: "integer",     pk: false, nullable: true,  note: "Prefix TipUni not Uni. V1 has FK to SAU_TIPUNI (not in GX XML — extra guard)." }
  - { gx: UniAtencaoSecundaria,  col: UniAtencaoSecundaria,  name: atencaoSecundaria,                   sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniBloqPacSemCadInd,   col: UniBloqPacSemCadInd,  name: bloquearPacienteSemCadastroIndividual, sql_type: "BOOLEAN",   pk: false, nullable: true }
  - { gx: UniAvisoVacinaAtrasada, col: UniAvisoVacinaAtrasada, name: avisoVacinaAtrasada,               sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniCadCPF,             col: UniCadCPF,             name: permitirCadastroPacienteSemCpf,      sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniPainel,             col: UniPainel,             name: usarPainelChamada,                   sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniRecInterMedMpp,     col: UniRecInterMedMpp,     name: emitirAlertasInteracaoMedicamentosa, sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniRecInterMedMppImp,  col: UniRecInterMedMppImp,  name: imprimirInteracaoMedicamentosa,      sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniBaiRemSemCns,       col: UniBaiRemSemCns,       name: permitirDispensacaoSemCns,           sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniBloqLancPcdAut,     col: UniBloqLancPcdAut,     name: bloquearLancamentoProcedimentoAutomatico, sql_type: "BOOLEAN", pk: false, nullable: true }
  - { gx: UniBloqDispPacExt,     col: UniBloqDispPacExt,     name: bloquearDispensacaoPacienteExterno,  sql_type: "BOOLEAN",     pk: false, nullable: true }
  - { gx: UniBloqAgSolExaPacExt, col: UniBloqAgSolExaPacExt, name: bloquearAgendaExamePacienteExterno,  sql_type: "BOOLEAN",     pk: false, nullable: true }

  # ── SAU_UNI1 (HIPERDIA — composite PK UniCod, UniProPesCod) ──────────────
  - { table: SAU_UNI1, gx: UniCod,       col: UniCod,       name: id.uniCod,       sql_type: "integer",  pk: true,  nullable: false }
  - { table: SAU_UNI1, gx: UniProPesCod, col: UniProPesCod, name: id.profissionalId, sql_type: "bigint", pk: true,  nullable: false, note: "FK → SAU_PRO (deferred Wave 4). FK check R56." }
  - { table: SAU_UNI1, gx: UniProDatInc, col: UniProDatInc, name: dataInclusao,    sql_type: "date",     pk: false, nullable: true,  note: "Required (R57)" }
  - { table: SAU_UNI1, gx: UniProMat,    col: UniProMat,    name: matricula,       sql_type: "CHAR(20)", pk: false, nullable: true }
  - { table: SAU_UNI1, gx: UniProCBO,    col: UniProCBO,    name: cbo,             sql_type: "VARCHAR(8)", pk: false, nullable: true }
  - { table: SAU_UNI1, gx: UniProSta,    col: UniProSta,    name: status,          sql_type: "smallint", pk: false, nullable: true,  note: "1=ATIVO, 2=INATIVO. Default 1 (R59). When 2, DatDes required (R58)." }
  - { table: SAU_UNI1, gx: UniProDatDes, col: UniProDatDes, name: dataDesativacao, sql_type: "date",     pk: false, nullable: true }

  # ── SAU_UNI2 (SISPRENATAL — composite PK UniCod, UniGesProPesCod, UniGesEspCod) ──
  - { table: SAU_UNI2, gx: UniCod,          col: UniCod,          name: id.uniCod,       sql_type: "integer", pk: true,  nullable: false }
  - { table: SAU_UNI2, gx: UniGesProPesCod, col: UniGesProPesCod, name: id.profissionalId, sql_type: "bigint", pk: true, nullable: false, note: "FK → SAU_PROESP (deferred). FK check R62,R64." }
  - { table: SAU_UNI2, gx: UniGesEspCod,    col: UniGesEspCod,    name: id.especialidadeId, sql_type: "integer", pk: true, nullable: false, note: "FK → SAU_ESP/SAU_PROESP. CBO code validated (R63-R66). SISPRENATAL CBO list (R65)." }
  - { table: SAU_UNI2, gx: UniGesDatInc,    col: UniGesDatInc,    name: dataInclusao,    sql_type: "date",    pk: false, nullable: true,  note: "Required (R67)" }
  - { table: SAU_UNI2, gx: UniGesSta,       col: UniGesSta,       name: status,          sql_type: "smallint", pk: false, nullable: true, note: "1=ATIVO,2=INATIVO. Default 1 (R69). When 2, DatDes required (R68)." }
  - { table: SAU_UNI2, gx: UniGesDatDes,    col: UniGesDatDes,    name: dataDesativacao, sql_type: "date",    pk: false, nullable: true }

  # ── SAU_UNI3 (Nutricionistas — composite PK UniCod, UniNutProPesCod, UniNutEspCod) ──
  - { table: SAU_UNI3, gx: UniCod,          col: UniCod,          name: id.uniCod,       sql_type: "integer", pk: true,  nullable: false }
  - { table: SAU_UNI3, gx: UniNutProPesCod, col: UniNutProPesCod, name: id.profissionalId, sql_type: "bigint", pk: true, nullable: false, note: "FK → SAU_PROESP (deferred). FK check R72,R75." }
  - { table: SAU_UNI3, gx: UniNutEspCod,    col: UniNutEspCod,    name: id.especialidadeId, sql_type: "integer", pk: true, nullable: false, note: "FK → SAU_ESP/SAU_PROESP. CBO validation (R73-R74)." }
  - { table: SAU_UNI3, gx: UniNutDatInc,    col: UniNutDatInc,    name: dataInclusao,    sql_type: "date",    pk: false, nullable: true,  note: "Required (R76)" }
  - { table: SAU_UNI3, gx: UniNutSta,       col: UniNutSta,       name: status,          sql_type: "smallint", pk: false, nullable: true, note: "1=ATIVO,2=INATIVO. When 2, DatDes required (R77). Default TBD — OQ6." }
  - { table: SAU_UNI3, gx: UniNutDatDes,    col: UniNutDatDes,    name: dataDesativacao, sql_type: "date",    pk: false, nullable: true }

  # ── SAU_UNISALA (Salas — composite PK UniCod, SalaCod) ────────────────────
  - { table: SAU_UNISALA, gx: UniCod,      col: UniCod,      name: id.uniCod,    sql_type: "integer",   pk: true,  nullable: false }
  - { table: SAU_UNISALA, gx: SalaCod,     col: SalaCod,     name: id.salaCodigo, sql_type: "smallint", pk: true,  nullable: false, note: "N(4,0) → smallint. Read-only after insert (R79)." }
  - { table: SAU_UNISALA, gx: SalaNom,     col: SalaNom,     name: nome,         sql_type: "VARCHAR(100)", pk: false, nullable: true }
  - { table: SAU_UNISALA, gx: SalaSta,     col: SalaSta,     name: status,       sql_type: "CHAR(1)",   pk: false, nullable: true,  note: "Single-char; values TBD — OQ9. Optimistic-lock tracked (R80)." }
  - { table: SAU_UNISALA, gx: SalaDatAlt,  col: SalaDatAlt,  name: dataAlteracao, sql_type: "TIMESTAMP WITHOUT TIME ZONE", pk: false, nullable: true, note: "Server auto-stamps to current datetime on insert/update (OQ1 confirmed)." }
  - { table: SAU_UNISALA, gx: SalaUsuLogin, col: SalaUsuLogin, name: usuarioLogin, sql_type: "VARCHAR(40)", pk: false, nullable: true, note: "Server auto-stamps to authenticated user login on insert/update (OQ1 confirmed)." }

  # ── Display-only (from lookups, not persisted) ────────────────────────────
  - { gx: UniMunNom,  name: municipioNome, stored: false, source: "SYS_MUN.MunNom (impl:4548-4555, R31)" }
  - { gx: UniMunUF,   name: municipioUf,   stored: false, source: "SYS_MUN.MunUF" }
  - { gx: UniMunIBGE, name: municipioIbge, stored: false, source: "SYS_MUN.MunIBGE" }

keys:
  pk:
    SAU_UNI:      [UniCod]
    SAU_UNI1:     [UniCod, UniProPesCod]
    SAU_UNI2:     [UniCod, UniGesProPesCod, UniGesEspCod]
    SAU_UNI3:     [UniCod, UniNutProPesCod, UniNutEspCod]
    SAU_UNISALA:  [UniCod, SalaCod]
  indexes:
    SAU_UNI:
      - { name: isau_uni4,     cols: [UniMunCod],        note: "missing from V1; added in V4" }
      - { name: usau_uni_desc, cols: [UniCod DESC],      note: "missing from V1; added in V4" }
      - { name: isau_uni1,     cols: [UniDisCod],        note: "missing from V1; added in V4" }
      - { name: isau_uni2,     cols: [UniProPesRespCod], note: "missing from V1; added in V4" }
      - { name: isau_uni6,     cols: [UniProPesDirCod],  note: "missing from V1; added in V4" }
      - { name: isau_uni7,     cols: [UniProPesAudCod],  note: "missing from V1; added in V4" }
      - { name: isau_uni8,     cols: [UniProPesAutCod],  note: "missing from V1; added in V4" }
    SAU_UNI1:
      - { name: idx_sau_uni1_propes, cols: [UniProPesCod],             note: "renamed from isau_uni13 to avoid clash with eventual FK constraint of same name (Wave 4)" }
    SAU_UNI2:
      - { name: idx_sau_uni2_proesp, cols: [UniGesProPesCod, UniGesEspCod], note: "renamed from isau_uni3" }
    SAU_UNI3:
      - { name: idx_sau_uni3_proesp, cols: [UniNutProPesCod, UniNutEspCod], note: "renamed from isau_uni5" }
  fks:
    # SAU_UNI outbound
    - { from: SAU_UNI, cols: [UniDisCod],      ref: SAU_DIS,    refCols: [DisCod],     constraint: fk_sau_uni_dis,    deferred: false, note: "in V1" }
    - { from: SAU_UNI, cols: [TipUniCod],      ref: SAU_TIPUNI, refCols: [TipUniCod],  constraint: fk_sau_uni_tipuni, deferred: false, note: "in V1; not in GX XML but safe" }
    - { from: SAU_UNI, cols: [UniMunCod],      ref: SYS_MUN,    refCols: [MunCod],     constraint: fk_sau_uni_mun,    deferred: false, note: "added in V4" }
    - { from: SAU_UNI, cols: [UniProPesRespCod], ref: SAU_PRO,  refCols: [ProPesCod],  constraint: isau_uni2,         deferred: true,  note: "add in Wave-4 migration" }
    - { from: SAU_UNI, cols: [UniProPesDirCod],  ref: SAU_PRO,  refCols: [ProPesCod],  constraint: isau_uni6,         deferred: true }
    - { from: SAU_UNI, cols: [UniProPesAudCod],  ref: SAU_PRO,  refCols: [ProPesCod],  constraint: isau_uni7,         deferred: true }
    - { from: SAU_UNI, cols: [UniProPesAutCod],  ref: SAU_PRO,  refCols: [ProPesCod],  constraint: isau_uni8,         deferred: true }
    # Sub-table outbound (non-deferred)
    - { from: SAU_UNI1, cols: [UniCod],                              ref: SAU_UNI,    refCols: [UniCod],            constraint: fk_sau_uni1_uni,         deferred: false }
    - { from: SAU_UNI2, cols: [UniCod],                              ref: SAU_UNI,    refCols: [UniCod],            constraint: fk_sau_uni2_uni,         deferred: false }
    - { from: SAU_UNI3, cols: [UniCod],                              ref: SAU_UNI,    refCols: [UniCod],            constraint: fk_sau_uni3_uni,         deferred: false }
    - { from: SAU_UNISALA, cols: [UniCod],                           ref: SAU_UNI,    refCols: [UniCod],            constraint: fk_sau_unisala_uni,      deferred: false }
    # Sub-table deferred
    - { from: SAU_UNI1, cols: [UniProPesCod],                        ref: SAU_PRO,    refCols: [ProPesCod],         constraint: isau_uni13,              deferred: true }
    - { from: SAU_UNI2, cols: [UniGesProPesCod, UniGesEspCod],       ref: SAU_PROESP, refCols: [ProPesCod, EspCod], constraint: isau_uni3,               deferred: true }
    - { from: SAU_UNI3, cols: [UniNutProPesCod, UniNutEspCod],       ref: SAU_PROESP, refCols: [ProPesCod, EspCod], constraint: isau_uni5,               deferred: true }

endpoints:
  - { method: GET,    path: /api/unidades,                                          desc: "list/search; filter by UniNom LIKE; paginated; includes CNES, sigla, situacao" }
  - { method: GET,    path: /api/unidades/{id},                                     desc: "full detail; includes derived municipioNome/UF/IBGE, professional display names; all sub-collections" }
  - { method: POST,   path: /api/unidades,                                          desc: "create; validates CNPJ (R12), required fields (R5-R11), FK existence, cross-field rules (R18-R26)" }
  - { method: PUT,    path: /api/unidades/{id},                                     desc: "update; UniNom stored UPPERCASE (R32); optimistic lock (R36)" }
  - { method: DELETE, path: /api/unidades/{id},                                     desc: "delete; 14 referential guards (R38-R51); cascade-deletes SAU_UNI1/2/3/UNISALA first (R52-R55)" }
  - { method: GET,    path: /api/unidades/lookup,                                   desc: "autocomplete (UniCod + UniNom); q= filter" }
  - { method: GET,    path: /api/unidades/{id}/hiperdia-profissionais,               desc: "list HIPERDIA professionals (SAU_UNI1)" }
  - { method: POST,   path: /api/unidades/{id}/hiperdia-profissionais,               desc: "add HIPERDIA professional; validates SAU_PRO FK (R56), DatInc required (R57), DatDes when INATIVO (R58)" }
  - { method: DELETE, path: /api/unidades/{id}/hiperdia-profissionais/{profId},      desc: "remove HIPERDIA professional" }
  - { method: GET,    path: /api/unidades/{id}/sisprenatal-profissionais,            desc: "list SISPRENATAL profs+especialidades (SAU_UNI2)" }
  - { method: POST,   path: /api/unidades/{id}/sisprenatal-profissionais,            desc: "add; validates SAU_PRO FK (R62), SAU_PROESP composite (R64), CBO (R65-R66), DatInc (R67)" }
  - { method: DELETE, path: /api/unidades/{id}/sisprenatal-profissionais/{pId}/{eId}, desc: "remove SISPRENATAL prof+especialidade" }
  - { method: GET,    path: /api/unidades/{id}/nutricionistas,                       desc: "list nutricionistas+especialidades (SAU_UNI3)" }
  - { method: POST,   path: /api/unidades/{id}/nutricionistas,                       desc: "add; validates SAU_PRO FK (R72), SAU_PROESP composite (R75), CBO (R73-R74), DatInc (R76)" }
  - { method: DELETE, path: /api/unidades/{id}/nutricionistas/{pId}/{eId},           desc: "remove nutricionista" }
  - { method: GET,    path: /api/unidades/{id}/salas,                                desc: "list salas (SAU_UNISALA)" }
  - { method: POST,   path: /api/unidades/{id}/salas,                                desc: "add sala; PK duplicate check (R79)" }
  - { method: PUT,    path: /api/unidades/{id}/salas/{salaCodigo},                   desc: "update sala; optimistic lock (R80)" }
  - { method: DELETE, path: /api/unidades/{id}/salas/{salaCodigo},                   desc: "delete sala" }

rules:
  # ── Authorization ──────────────────────────────────────────────────────────
  - id: R1
    when: always
    desc: "Session required; empty user context → redirect hnotauthorizeduser; invalid permission for mode → redirect hnotauthorizedprompt."
    source: "sau_uni_impl.java:3882-3907"
    confidence: high
    test: UnidadeServiceTest#rejectsUnauthenticatedAccess

  # ── Defaults on insert ────────────────────────────────────────────────────
  - id: R2
    when: insert
    desc: "UniGes defaults to 1 (MUNICIPAL)."
    source: "sau_uni_impl.java:11249, 4206-4211"
    confidence: high
    test: UnidadeServiceTest#defaultsUniGesToMunicipal

  - id: R3
    when: insert
    desc: "UniSit defaults to 1 (ATIVADO)."
    source: "sau_uni_impl.java:11252, 4212-4217"
    confidence: high
    test: UnidadeServiceTest#defaultsUniSitToAtivado

  - id: R4
    when: insert
    desc: "UniExt defaults to false."
    source: "sau_uni_impl.java:11210"
    confidence: high
    test: UnidadeServiceTest#defaultsUniExtToFalse

  # ── Required fields ───────────────────────────────────────────────────────
  - id: R5
    when: insert|update
    desc: "UniNom required; blank → 'Informe o Nome da Unidade!'"
    source: "sau_uni_impl.java:4472-4478"
    confidence: high
    test: UnidadeServiceTest#rejectsBlankUniNom

  - id: R6
    when: insert|update
    desc: "UniCnpj required; blank → 'Informe o CNPJ!'"
    source: "sau_uni_impl.java:4484-4490"
    confidence: high
    test: UnidadeServiceTest#rejectsBlankUniCnpj

  - id: R7
    when: insert|update
    desc: "UniCep required; blank → 'Informe o CEP!'"
    source: "sau_uni_impl.java:4508-4513"
    confidence: high
    test: UnidadeServiceTest#rejectsBlankUniCep

  - id: R8
    when: insert|update
    desc: "UniEnd required; blank → 'Informe o endereço!'"
    source: "sau_uni_impl.java:4515-4520"
    confidence: high
    test: UnidadeServiceTest#rejectsBlankUniEnd

  - id: R9
    when: insert|update
    desc: "UniEndNum required; blank → 'Informe o número do endereço!'"
    source: "sau_uni_impl.java:4522-4528"
    confidence: high
    test: UnidadeServiceTest#rejectsBlankUniEndNum

  - id: R10
    when: insert|update
    desc: "UniBai required; blank → 'Informe o Bairro!'"
    source: "sau_uni_impl.java:4529-4535"
    confidence: high
    test: UnidadeServiceTest#rejectsBlankUniBai

  - id: R11
    when: insert|update
    desc: "UniMunCod must be non-zero; zero → 'Informe o código do Município!'"
    source: "sau_uni_impl.java:4557-4563"
    confidence: high
    test: UnidadeServiceTest#rejectsZeroUniMunCod

  # ── Format validations ────────────────────────────────────────────────────
  - id: R12
    when: insert|update
    desc: "UniCnpj must pass the psau_val_cnpjcpf check-digit algorithm (same as SAU_PAC). Invalid CNPJ → 'CNPJ inválido! Tente novamente.'"
    source: "sau_uni_impl.java:4491-4507, 44-58"
    confidence: high
    test: UnidadeServiceTest#rejectsInvalidCnpj

  - id: R13
    when: insert|update
    desc: "UniFon, when non-blank, must match Brazilian phone regex (DDD) NNNNN-NNNN. Invalid → 'Telefone inválido!'"
    source: "sau_uni_impl.java:4564-4569"
    confidence: high
    test: UnidadeServiceTest#rejectsInvalidUniFon

  - id: R14
    when: insert|update
    desc: "UniFax, when non-blank, must match same Brazilian phone regex. Invalid → 'Celular inválido!'"
    source: "sau_uni_impl.java:4571-4576"
    confidence: high
    test: UnidadeServiceTest#rejectsInvalidUniFax

  # ── Numeric range validations ─────────────────────────────────────────────
  - id: R15
    when: insert|update
    desc: "UniCnes must be in range 0–9999999."
    source: "sau_uni_impl.java:2676-2685"
    confidence: high
    test: UnidadeServiceTest#rejectsCnesOutOfRange

  - id: R16
    when: insert|update
    desc: "UniMunCod must be in range 0–999999999."
    source: "sau_uni_impl.java:2720-2729"
    confidence: high
    test: UnidadeServiceTest#rejectsMunCodOutOfRange

  - id: R17
    when: insert|update
    desc: "UniForPesCod must be in range 0–99999999999999."
    source: "sau_uni_impl.java:2882-2891"
    confidence: high
    test: UnidadeServiceTest#rejectsForPesCodOutOfRange

  # ── Cross-field rules ─────────────────────────────────────────────────────
  - id: R18
    when: insert|update
    desc: "UniCnes must be non-zero when any of UniBPA=1, UniSisPreNatal=1, UniHiperdia=1, or UniSIPNI=1. Missing CNES → 'Informe o número do CNES!'"
    source: "sau_uni_impl.java:4578-4584"
    confidence: high
    test: UnidadeServiceTest#requiresCnesWhenProgramFlagsSet

  - id: R19
    when: insert|update
    desc: "When UniOrgEmi is non-blank, UniProPesDirCod (Diretor Clínico) must be provided. Missing → 'Informe o Diretor Clínico!'"
    source: "sau_uni_impl.java:4585-4591"
    confidence: high
    test: UnidadeServiceTest#requiresDirCodWhenOrgEmiPresent

  - id: R20
    when: insert|update
    desc: "When UniOrgEmi is non-blank, UniProPesAutCod (Autorizador) must be provided. Missing → 'Informe o Profissional Auditor!'"
    source: "sau_uni_impl.java:4592-4598"
    confidence: high
    test: UnidadeServiceTest#requiresAutCodWhenOrgEmiPresent

  # ── FK existence checks ───────────────────────────────────────────────────
  - id: R21
    when: insert|update
    desc: "UniMunCod must exist in SYS_MUN. Not found → 'Não existe Município.'"
    source: "sau_uni_impl.java:4537-4556"
    confidence: high
    test: UnidadeServiceTest#rejectsNonExistentMunicipio

  - id: R22
    when: insert|update
    desc: "UniProPesDirCod must exist in SAU_PRO when non-zero. Not found → 'Não existe DIRETOR CLINICO.'"
    source: "sau_uni_impl.java:4599-4634"
    confidence: high
    test: UnidadeServiceTest#rejectsNonExistentDirCod

  - id: R23
    when: insert|update
    desc: "UniProPesDirCod, when provided, must have a non-blank CNS on SAU_PRO. Missing → 'Informe o CNS do Diretor Clínico!'"
    source: "sau_uni_impl.java:4635-4641"
    confidence: high
    test: UnidadeServiceTest#requiresCnsDirCod

  - id: R24
    when: insert|update
    desc: "UniProPesAutCod must exist in SAU_PRO when non-zero. Not found → 'Não existe Profissional Autorizador.'"
    source: "sau_uni_impl.java:4642-4683"
    confidence: high
    test: UnidadeServiceTest#rejectsNonExistentAutCod

  - id: R25
    when: insert|update
    desc: "UniProPesAutCod, when provided, must have a non-blank CNS on SAU_PRO. Missing → 'Informe o CNS do Profissional Auditor!'"
    source: "sau_uni_impl.java:4685-4691"
    confidence: high
    test: UnidadeServiceTest#requiresCnsAutCod

  - id: R26
    when: insert|update
    desc: "Autorizador and Diretor Clínico must be different people (different CNS), UNLESS UniOrgEmi starts with 'U' or 'S'. Violation → 'Profissional Autorizador deve ser diferente do Diretor Clínico!'"
    source: "sau_uni_impl.java:4660-4666"
    confidence: high
    test: UnidadeServiceTest#rejectsSameAutorizadorAndDiretor
    notes: "Exception for OrgEmi starting 'U'/'S' — meaning TBD, see OQ3"

  - id: R27
    when: insert|update
    desc: "UniProPesRespCod must exist in SAU_PRO when non-zero. Not found → 'Não existe MÉDICO RESPONSÁVEL.'"
    source: "sau_uni_impl.java:4721-4749"
    confidence: high
    test: UnidadeServiceTest#rejectsNonExistentRespCod

  - id: R28
    when: insert|update
    desc: "UniProPesAudCod must exist in SAU_PRO when non-zero. Not found → 'Não existe Profissional Auditor.'"
    source: "sau_uni_impl.java:4751-4779"
    confidence: high
    test: UnidadeServiceTest#rejectsNonExistentAudCod

  - id: R29
    when: insert|update
    desc: "UniDisCod must exist in SAU_DIS when non-zero. Not found → 'Não existe DISTRITO.'"
    source: "sau_uni_impl.java:4692-4706"
    confidence: high
    test: UnidadeServiceTest#rejectsNonExistentDistrito

  # ── Derived fields ────────────────────────────────────────────────────────
  - id: R30
    when: load
    desc: "A928uniIdENome (display formula) = trim(UniCod) + ' - ' + trim(UniNom). Not persisted; returned in response DTO."
    source: "sau_uni_impl.java:4443, 4470, 6091"
    confidence: high
    test: UnidadeServiceTest#computesUniIdENome

  - id: R31
    when: load
    desc: "On load, UniMunNom, UniMunUF, UniMunIBGE are populated from SYS_MUN via lookup; not stored in SAU_UNI."
    source: "sau_uni_impl.java:4548-4555"
    confidence: high
    test: UnidadeServiceTest#derivesMunNomFromMunCod

  # ── Uppercase transforms (server-side) ────────────────────────────────────
  - id: R32
    when: insert|update
    desc: "UniNom stored UPPERCASE (GXutil.upper before persistence)."
    source: "sau_uni_impl.java:2673"
    confidence: high
    test: UnidadeServiceTest#storesUniNomUpperCase

  - id: R33
    when: insert|update
    desc: "UniSigla stored UPPERCASE."
    source: "sau_uni_impl.java:2699"
    confidence: high
    test: UnidadeServiceTest#storesUniSiglaUpperCase

  - id: R34
    when: insert|update
    desc: "UniRazSoc stored UPPERCASE."
    source: "sau_uni_impl.java:2703"
    confidence: high
    test: UnidadeServiceTest#storesUniRazSocUpperCase

  - id: R35
    when: insert|update
    desc: "UniEnd, UnEndCom, UniBai, UniMunNom, UniRes, UniOrgEmi all stored UPPERCASE."
    source: "sau_uni_impl.java:2705-2715, 2737, 2749, 2899"
    confidence: high
    test: UnidadeServiceTest#storesAddressFieldsUpperCase

  # ── Optimistic concurrency ────────────────────────────────────────────────
  - id: R36
    when: update|delete
    desc: "All 55 SAU_UNI columns compared against load-time snapshot; any change → GXM_waschg ('SAU_UNI'). Implement via @Version or ETag."
    source: "sau_uni_impl.java:5490-5873"
    confidence: high
    test: UnidadeServiceTest#rejectsConcurrentModification

  - id: R37
    when: update|delete
    desc: "Row locked by another session (DB status 103) → GXM_lock ('SAU_UNI')."
    source: "sau_uni_impl.java:5496-5500"
    confidence: high
    test: UnidadeServiceTest#rejectsLockedRecord

  # ── Delete guards ─────────────────────────────────────────────────────────
  - id: R38
    when: delete
    desc: "Block if 'Dias da semana' row references this unidade."
    source: "sau_uni_impl.java:6192-6198"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenDiasSemanaExists

  - id: R39
    when: delete
    desc: "Block if referenced as Unidade Fornecedora in Configuração Saldo de Medicamento."
    source: "sau_uni_impl.java:6200-6206"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenSaldoMedFornExists

  - id: R40
    when: delete
    desc: "Block if referenced as Unidade Solicitante in Configuração Saldo de Medicamento."
    source: "sau_uni_impl.java:6207-6213"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenSaldoMedSolExists

  - id: R41
    when: delete
    desc: "Block if any Usuario/Unidade association exists."
    source: "sau_uni_impl.java:6215-6222"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenUsuarioUnidadeExists

  - id: R42
    when: delete
    desc: "Block if any Usuário do Sistema references this unidade."
    source: "sau_uni_impl.java:6223-6230"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenUsuarioSistemaExists

  - id: R43
    when: delete
    desc: "Block if any Unidade do Medicamento (SAU_REM1) references this unidade."
    source: "sau_uni_impl.java:6231-6238"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenUnidadeMedicamentoExists

  - id: R44
    when: delete
    desc: "Block if any SAU_REM_UNISETOR row references this unidade."
    source: "sau_uni_impl.java:6239-6246"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenRemUniSetorExists

  - id: R45
    when: delete
    desc: "Block if any Paciente references this unidade as their atualização de cadastro unit."
    source: "sau_uni_impl.java:6247-6254"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenPacienteAtualizacaoExists

  - id: R46
    when: delete
    desc: "Block if any Paciente references this unidade as their inclusão de cadastro unit."
    source: "sau_uni_impl.java:6255-6262"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenPacienteInclusaoExists

  - id: R47
    when: delete
    desc: "Block if any Paciente has this as their main Unidade."
    source: "sau_uni_impl.java:6263-6270"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenPacienteUnidadeExists

  - id: R48
    when: delete
    desc: "Block if any production record references this as Unidade de destino da produção."
    source: "sau_uni_impl.java:6271-6278"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenProducaoDestExists

  - id: R49
    when: delete
    desc: "Block if any scheduling record references this as Unidade do Agendamento Externo."
    source: "sau_uni_impl.java:6279-6286"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenAgendamentoExtExists

  - id: R50
    when: delete
    desc: "Block if any Setor/Departamento (SAU_UNISETOR) references this unidade."
    source: "sau_uni_impl.java:6287-6294"
    confidence: high
    test: UnidadeServiceTest#deleteForbiddenWhenSetorExists

  - id: R51
    when: delete
    desc: "Block if any Receituário Controle Especial (SAU_RECESP) references this unidade. Regulatory guard."
    source: "sau_uni_impl.java:6295-6302"
    confidence: medium
    test: UnidadeServiceTest#deleteForbiddenWhenReceitaEspecialExists
    notes: "Verify with regulatory lead (OQ10)"

  # ── Cascade deletes ───────────────────────────────────────────────────────
  - id: R52
    when: delete
    desc: "Cascade-delete all SAU_UNISALA rows before deleting the parent SAU_UNI row."
    source: "sau_uni_impl.java:6014-6021"
    confidence: high
    test: UnidadeServiceTest#cascadeDeletesSalas

  - id: R53
    when: delete
    desc: "Cascade-delete all SAU_UNI3 (nutricionistas) rows before deleting the parent."
    source: "sau_uni_impl.java:6022-6029"
    confidence: high
    test: UnidadeServiceTest#cascadeDeletesNutricionistas

  - id: R54
    when: delete
    desc: "Cascade-delete all SAU_UNI2 (SISPRENATAL) rows before deleting the parent."
    source: "sau_uni_impl.java:6030-6037"
    confidence: high
    test: UnidadeServiceTest#cascadeDeletesSisPreNatalProfs

  - id: R55
    when: delete
    desc: "Cascade-delete all SAU_UNI1 (HIPERDIA) rows before deleting the parent."
    source: "sau_uni_impl.java:6038-6045"
    confidence: high
    test: UnidadeServiceTest#cascadeDeletesHiperdiaProfs

  # ── SAU_UNI1 rules ────────────────────────────────────────────────────────
  - id: R56
    when: insert|update
    desc: "SAU_UNI1: UniProPesCod must exist in SAU_PRO. Not found → 'Não existe PROFISSONAL.'"
    source: "sau_uni_impl.java:7056-7080"
    confidence: high
    test: UnidadeServiceTest#uni1RejectsNonExistentProfissional

  - id: R57
    when: insert|update
    desc: "SAU_UNI1: UniProDatInc required. Null → 'Informe a Data de Inclusão!'"
    source: "sau_uni_impl.java:7089-7096"
    confidence: high
    test: UnidadeServiceTest#uni1RequiresDatInc

  - id: R58
    when: insert|update
    desc: "SAU_UNI1: When UniProSta=2 (INATIVO), UniProDatDes required. Missing → 'Informe a Data de Desativação!'"
    source: "sau_uni_impl.java:7097-7104"
    confidence: high
    test: UnidadeServiceTest#uni1RequiresDatDesWhenInativo

  - id: R59
    when: insert
    desc: "SAU_UNI1: UniProSta defaults to 1 (ATIVO)."
    source: "sau_uni_impl.java:6996-6999"
    confidence: high
    test: UnidadeServiceTest#uni1DefaultsProStaToAtivo

  - id: R60
    when: insert
    desc: "SAU_UNI1: Duplicate PK (UniCod + UniProPesCod) → DuplicatePrimaryKey (GXM_noupdate)."
    source: "sau_uni_impl.java:7231-7235"
    confidence: medium
    test: UnidadeServiceTest#uni1RejectsDuplicatePk

  - id: R61
    when: update|delete
    desc: "SAU_UNI1: Optimistic-concurrency check; stale data → GXM_waschg ('SAU_UNI1')."
    source: "sau_uni_impl.java: checkOptimisticConcurrency0E25 (pattern confirmed from 0E26 at :7984)"
    confidence: medium
    test: UnidadeServiceTest#uni1RejectsConcurrentModification
    notes: "Exact columns checked — OQ5"

  # ── SAU_UNI2 rules ────────────────────────────────────────────────────────
  - id: R62
    when: insert|update
    desc: "SAU_UNI2: UniGesProPesCod must exist in SAU_PRO. Not found → 'Não existe Profissional.'"
    source: "sau_uni_impl.java:7671-7701"
    confidence: high
    test: UnidadeServiceTest#uni2RejectsNonExistentProfissional

  - id: R63
    when: insert|update
    desc: "SAU_UNI2: UniGesEspCod must reference an existing SAU_ESP. Error message reuses 'Não existe Profissional' (likely a GX copy-paste — see OQ4)."
    source: "sau_uni_impl.java:7703-7717"
    confidence: high
    test: UnidadeServiceTest#uni2RejectsNonExistentEspecialidade

  - id: R64
    when: insert|update
    desc: "SAU_UNI2: (UniGesProPesCod, UniGesEspCod) must reference an existing SAU_PROESP row."
    source: "sau_uni_impl.java:7737-7745"
    confidence: high
    test: UnidadeServiceTest#uni2RejectsNonExistentProEsp

  - id: R65
    when: insert|update
    desc: "SAU_UNI2: CBO code derived from the especialidade must be one of 7 allowed SISPRENATAL codes: 223505, 225125, 225250, 225142, 223565, 223545, 225170. Violation → 'CBO inválido para o SISPRENATAL!'"
    source: "sau_uni_impl.java:7731-7735"
    confidence: high
    test: UnidadeServiceTest#uni2RejectsInvalidCboForSisPreNatal
    notes: "Verify CBO list is current — OQ8"

  - id: R66
    when: insert|update
    desc: "SAU_UNI2: CBO code from especialidade must exist in the CBO reference table when non-blank. Not found → 'Não existe CBO.'"
    source: "sau_uni_impl.java:7718-7730"
    confidence: high
    test: UnidadeServiceTest#uni2RejectsNonExistentCbo

  - id: R67
    when: insert|update
    desc: "SAU_UNI2: UniGesDatInc required. Null → 'Informe a Data de Inclusão!'"
    source: "sau_uni_impl.java:7747-7754"
    confidence: high
    test: UnidadeServiceTest#uni2RequiresDatInc

  - id: R68
    when: insert|update
    desc: "SAU_UNI2: When UniGesSta=2 (INATIVO), UniGesDatDes required."
    source: "sau_uni_impl.java:7755-7762"
    confidence: high
    test: UnidadeServiceTest#uni2RequiresDatDesWhenInativo

  - id: R69
    when: insert
    desc: "SAU_UNI2: UniGesSta defaults to 1 (ATIVO)."
    source: "sau_uni_impl.java:7604-7607"
    confidence: high
    test: UnidadeServiceTest#uni2DefaultsGesStaToAtivo

  - id: R70
    when: insert
    desc: "SAU_UNI2: Duplicate PK (UniCod + UniGesProPesCod + UniGesEspCod) → DuplicatePrimaryKey."
    source: "sau_uni_impl.java: insert0E26 cursor T000E93"
    confidence: medium
    test: UnidadeServiceTest#uni2RejectsDuplicatePk

  - id: R71
    when: update|delete
    desc: "SAU_UNI2: Optimistic-concurrency check covers UniGesSta, UniGesDatInc, UniGesDatDes. Stale → GXM_waschg ('SAU_UNI2')."
    source: "sau_uni_impl.java:7984-8001"
    confidence: high
    test: UnidadeServiceTest#uni2RejectsConcurrentModification

  # ── SAU_UNI3 rules ────────────────────────────────────────────────────────
  - id: R72
    when: insert|update
    desc: "SAU_UNI3: UniNutProPesCod must exist in SAU_PRO. Not found → 'Não existe Profissional.'"
    source: "sau_uni_impl.java:8402-8428"
    confidence: high
    test: UnidadeServiceTest#uni3RejectsNonExistentNutricionista

  - id: R73
    when: insert|update
    desc: "SAU_UNI3: UniNutEspCod must reference SAU_ESP. Not found → 'Não existe Profissional.'"
    source: "sau_uni_impl.java:8429-8443"
    confidence: high
    test: UnidadeServiceTest#uni3RejectsNonExistentEspecialidade

  - id: R74
    when: insert|update
    desc: "SAU_UNI3: CBO code from especialidade must exist in CBO table when non-blank. Not found → 'Não existe CBO.'"
    source: "sau_uni_impl.java:8444-8456"
    confidence: high
    test: UnidadeServiceTest#uni3RejectsNonExistentCbo

  - id: R75
    when: insert|update
    desc: "SAU_UNI3: (UniNutProPesCod, UniNutEspCod) must reference existing SAU_PROESP row."
    source: "sau_uni_impl.java:8457-8466"
    confidence: high
    test: UnidadeServiceTest#uni3RejectsNonExistentProEsp

  - id: R76
    when: insert|update
    desc: "SAU_UNI3: UniNutDatInc required. Null → 'Informe a Data de Inclusão!'"
    source: "sau_uni_impl.java:8468-8475"
    confidence: high
    test: UnidadeServiceTest#uni3RequiresDatInc

  - id: R77
    when: insert|update
    desc: "SAU_UNI3: When UniNutSta=2 (INATIVO), UniNutDatDes required."
    source: "sau_uni_impl.java:8476-8483"
    confidence: high
    test: UnidadeServiceTest#uni3RequiresDatDesWhenInativo

  - id: R78
    when: insert
    desc: "SAU_UNI3: Duplicate PK (UniCod + UniNutProPesCod + UniNutEspCod) → DuplicatePrimaryKey."
    source: "sau_uni_impl.java: insert0E27"
    confidence: medium
    test: UnidadeServiceTest#uni3RejectsDuplicatePk

  # ── SAU_UNISALA rules ─────────────────────────────────────────────────────
  - id: R79
    when: insert
    desc: "SAU_UNISALA: PK (UniCod + SalaCod) must be unique. SalaCod read-only after insert. Duplicate → DuplicatePrimaryKey."
    source: "sau_uni_impl.java:9050-9057, 9231-9234"
    confidence: high
    test: UnidadeServiceTest#salaRejectsDuplicatePk

  - id: R80
    when: update|delete
    desc: "SAU_UNISALA: Optimistic-concurrency check on SalaNom, SalaSta, SalaDatAlt, SalaUsuLogin. Stale → GXM_waschg ('SAU_UNISALA')."
    source: "sau_uni_impl.java:9164-9206"
    confidence: high
    test: UnidadeServiceTest#salaRejectsConcurrentModification

  - id: R81
    when: insert|update
    desc: "SAU_UNISALA: SalaSta is a single CHAR(1) field; no explicit enum validation found in checkExtendedTable0E28 (empty). Allowed values must be confirmed from DB or domain spec — OQ9."
    source: "sau_uni_impl.java:9085-9090"
    confidence: low
    test: UnidadeServiceTest#salaAcceptsValidSalaSta

  - id: R82
    when: insert|update
    desc: "SAU_UNISALA: SalaDatAlt auto-stamped to current datetime; SalaUsuLogin auto-stamped to authenticated user login. Never trust form-submitted values for these two columns."
    source: "sau_uni_impl.java:9228-9229 (confirmed OQ1 2026-06-21)"
    confidence: high
    test: UnidadeServiceTest#salaStampsDateAndUser

  # ── UniExt section visibility ─────────────────────────────────────────────
  - id: R83
    when: load|insert|update
    desc: "External-configuration section hidden when UniExt=false; shown when UniExt=true. External-patient policy fields within that section are meaningful only when UniExt=true."
    source: "sau_uni_impl.java:4450-4462, 4708-4720"
    confidence: high
    test: UnidadeServiceTest#hidesExternalSectionWhenUniExtFalse

  # ── PK duplicate ──────────────────────────────────────────────────────────
  - id: R84
    when: insert
    desc: "SAU_UNI: Duplicate UniCod → DuplicatePrimaryKey (GXM_noupdate)."
    source: "sau_uni_impl.java:5898-5902"
    confidence: high
    test: UnidadeServiceTest#rejectsDuplicatePk

phi_fields:
  - { field: UniCnpj,           reason: "Corporate identifier of the health facility (CNPJ); regulated health-establishment identifier in CNES ecosystem. Do not log." }
  - { field: UniCnes,           reason: "National health establishment code; appears in patient care records. Operationally sensitive." }
  - { field: UniRes,            reason: "Nome do Responsável — natural person name; LGPD Art. 5 I. @Auditable on reads." }
  - { field: UniProPesRespCod,  reason: "FK to SAU_PRO; derived responses include professional name + CPF/CNPJ. @Auditable." }
  - { field: UniProPesDirCod,   reason: "FK to SAU_PRO — same as above." }
  - { field: UniProPesAudCod,   reason: "FK to SAU_PRO — same as above." }
  - { field: UniProPesAutCod,   reason: "FK to SAU_PRO — same as above." }
# Note: policy flags (UniCadCNS, UniCadCPF, UniAteSemCNS etc.) govern LGPD-relevant
# patient-data collection rules. Wrong migration of these flags causes systemic PHI failures.

auth:
  roles_required:
    read:             [SAUDE_CADASTRO]
    write_sub_tables: [SAUDE_CADASTRO]   # UNI1/2/3/UNISALA — routine staff/room management
    write_main:       [SAUDE_ADMIN]      # SAU_UNI CREATE/UPDATE/DELETE — confirmed OQ10 2026-06-21
  notes: >
    GeneXus uses session-cookie guard (ploadcontext + pisauthorized pattern, impl:3882-3907)
    not IntegratedSecurity (IntegratedSecurityEnabled=false at sau_uni.java:33-35).
    Spring mapping: use @PreAuthorize("hasRole('SAUDE_ADMIN')") on POST/PUT/DELETE /api/unidades
    and @PreAuthorize("hasRole('SAUDE_CADASTRO')") on sub-resource endpoints and all GETs.
    SAUDE_ADMIN required for main-table writes because the 21 Boolean policy flags (UniCadCNS,
    UniAteSemCNS, UniBloqPacSemCadInd, UniRecInterMedMpp, etc.) control system-wide patient-intake
    rules — a wrong value affects every clinic operator.
    Verify exact SAU_PRFCON permission codes before cutover:
      SELECT * FROM SAU_PRFCON WHERE PrgNom IN ('SAU_UNI', 'HPROMPTSAU_UNI');

parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios:
    - "GET /api/unidades — list returns nome, cnes, sigla, situacao"
    - "GET /api/unidades/{id} — all 55 columns + derived municipioNome/UF/IBGE"
    - "GET /api/unidades/lookup?q=CENTRO — autocomplete"
    - "POST valid minimum — UniCod + UniNom + UniCnpj + UniCep + UniEnd + UniEndNum + UniBai + UniMunCod → 201"
    - "POST blank UniNom → 422 with 'Informe o Nome da Unidade!'"
    - "POST invalid CNPJ → 422 with 'CNPJ inválido!'"
    - "POST invalid phone → 422 with 'Telefone inválido!'"
    - "POST UniHiperdia=1 without UniCnes → 422 'Informe o número do CNES!'"
    - "POST UniOrgEmi set without DirCod → 422 'Informe o Diretor Clínico!'"
    - "POST DirCod=AutCod (same person, OrgEmi≠U/S) → 422"
    - "PUT UniNom — verify stored UPPERCASE"
    - "PUT change UniSit to 2 (DESATIVADO)"
    - "DELETE unidade with no children → 204"
    - "DELETE with SAU_UNISETOR children → 409"
    - "DELETE with SAU_USUUNI references → 409"
    - "DELETE with SAU_REM1 reference → 409"
    - "POST /api/unidades/{id}/hiperdia-profissionais valid → 201"
    - "POST /api/unidades/{id}/hiperdia-profissionais without DatInc → 422"
    - "POST /api/unidades/{id}/sisprenatal-profissionais invalid CBO → 422 'CBO inválido para o SISPRENATAL!'"
    - "POST /api/unidades/{id}/salas duplicate SalaCod → 409"

status: verified

# Backend + frontend + tests generated, reviewed, and parity-verified on 2026-06-21 (/migrate-slice + /verify-parity SAU_UNI).
# Unit: UnidadeServiceTest 27/27 + UnidadeSubServiceTest 4/4. Integration: full mvn verify green (193 IT, 0 fail).
# Parity: 18/20 PARITY vs shared GeneXus snapshot (DB-state method); 2 DIVERGE, both documented & accepted.
parity_result:
  date: 2026-06-21
  method: "DB-state comparison; new app :8090 vs shared non-prod snapshot saude-mandaguari (140 SAU_UNI rows); legacy Tomcat reachable at host.docker.internal:8080"
  report: "backend/src/test/resources/parity/unidade/PARITY-REPORT.md (+ scenarios.json fixtures)"
  summary: "18/20 PARITY"
  parity: [1,2,3,4,6,7,8,9,10,11,12,13,14,15,16,17,18,20]
  diverge:
    - { scenario: 5,  rule: RF6, detail: "blank UniNom → 400 (@NotBlank) instead of 422+legacy message; field still rejected", disposition: accepted }
    - { scenario: 19, rule: RF2, detail: "SISPRENATAL invalid CBO → 201 (CBO validation deferred to Wave 4) instead of 422", disposition: "known deferral — cutover caveat for SISPRENATAL sub-resource only" }
  bug_fixed: "Duplicate SalaCod silently upserted (201). UnidadeSubService.addSala now guards existsById → 409. Covered by UnidadeSubServiceTest#addSalaRejectsDuplicateSalaCod. (scenario #20 → PARITY)"
  notes:
    - "#2 detail read is PARITY on persisted columns but still lacks derived municipioNome/UF/IBGE + uniIdENome (RF4)."
    - "Verified with Flyway disabled + ddl-auto=none (schema already present in the live copy). seq_sau_uni_cod was created manually in the snapshot (V4-equivalent, idempotent)."
    - "All synthetic rows used the ZZPARITY marker and were deleted; 0 residual rows after the run."

code_review:
  reviewer: migration-reviewer
  date: 2026-06-21
  resolved:
    - "UniEsfAdm clean name corrected estrategiaFamiliar → esferaAdministrativa (entity/DTO/service/frontend). Physical column UniEsfAdm unchanged."
    - "Auth gate proven: SAUDE_ADMIN enforced for POST/PUT/DELETE /api/unidades; added IT createForbiddenForCadastroRole/deleteForbiddenForCadastroRole (403) + getAllowedForCadastroRole (200). Fixed pre-existing IT that wrote as SAUDE_CADASTRO."
    - "LGPD: PHI read-audit added to UnidadeService.get() (exposes responsavel + professional FKs)."
    - "R4 default externo=false, R16 sequence UniCod, R26 autorizador≠diretor (+U/S exception), R32–R35 uppercase, R52–R55 cascade delete, R82 Sala auto-stamp all implemented + unit-tested."
    - "IT @BeforeEach cleanup now truncates SAU_UNISALA/UNI1/UNI2/UNI3 before SAU_UNI."
  deferred:
    - id: RF1
      finding: "Optimistic locking R36/R37 (main) and R80 (Sala) not implemented — no @Version. Last-writer-wins."
      action: "Add @Version + ETag handling in a follow-up; add concurrency tests."
    - id: RF2
      finding: "SISPRENATAL/nutricionista CBO allow-list (R65/R66) and ESP-existence (R63/R73) reduced to null/zero guards — real SAU_ESP/SAU_PROESP lookups deferred to Wave 4."
      action: "Implement lookups when SAU_PRO/PROESP migrate; until then invalid CBO/ESP can persist."
    - id: RF3
      finding: "No UnidadeSubServiceTest — sub-table rules R56–R82 are unit-untested; sub-resource endpoints have no IT."
      action: "Author UnidadeSubServiceTest + sub-resource IT in a follow-up pass."
    - id: RF4
      finding: "Derived fields R30 (uniIdENome) and R31 (municipioNome/UF/IBGE, professional display names) not in UnidadeResponse/mapper."
      action: "Add derived fields with read-audit on professional names; needs SYS_MUN/SAU_PRO joins."
    - id: RF5
      finding: "Parity suite is @Disabled stubs (20 scenarios). Golden-master fixtures not yet captured."
      action: "Gate cutover on /verify-parity SAU_UNI (requires running legacy app)."
    - id: RF6
      finding: "Validation HTTP status inconsistent: @NotBlank → 400 vs BusinessRule → 422; legacy emits uniform 422 with Portuguese message."
      action: "Confirm whether to route required-field failures through BusinessRule→422 for parity."

open_questions:
  - id: OQ1
    topic: SAU_UNISALA audit stamp
    status: RESOLVED 2026-06-21
    resolution: "Server auto-stamps SalaDatAlt=now() and SalaUsuLogin=authenticatedUser on every insert/update. R82 upgraded to high."
    rules: [R82]

  - id: OQ2
    topic: UniProPesRespCod / UniProPesAudCod optionality
    desc: "R27 and R28 check FK existence but there is no required-non-zero rule for these two professional FKs. Confirm whether they are truly optional or must be provided in certain conditions."
    rules: [R27, R28]

  - id: OQ3
    topic: UniOrgEmi 'U'/'S' exception
    desc: "R26: Autorizador=Diretor exception when OrgEmi starts with 'U' or 'S'. What do U and S represent? Verify with domain expert before implementing the exception path."
    rules: [R26]

  - id: OQ4
    topic: R63 error message
    desc: "SAU_UNI2: especialidade FK error message says 'Não existe Profissional' (same as the profissional check). Confirm if this is a GeneXus KB copy-paste bug or intentional — if a bug, fix the error text in the new service."
    rules: [R63]

  - id: OQ5
    topic: SAU_UNI1 optimistic concurrency columns
    desc: "R61: checkOptimisticConcurrency0E25 was not fully read; which columns does it compare? Expected: UniProSta, UniProDatInc, UniProDatDes, UniProMat, UniProCBO."
    rules: [R61]

  - id: OQ6
    topic: SAU_UNI3 UniNutSta default
    desc: "R72+: standaloneModal0E27 not fully read. Does UniNutSta default to 1 (ATIVO) on insert, matching SAU_UNI1 (R59) and SAU_UNI2 (R69)?"

  - id: OQ7
    topic: Audit logging
    desc: "No psau_inc_log / SAU_LOG audit calls found in SAU_UNI. Is SAU_UNI audited by a separate trigger, calling transaction, or not at all? The Spring service must add @Auditable regardless (LGPD)."

  - id: OQ8
    topic: R65 SISPRENATAL CBO codes
    desc: "The 7 CBO codes hardcoded in R65 (223505, 225125, 225250, 225142, 223565, 223545, 225170) must be verified against current SISPRENATAL/CNES regulation — these could have changed since the GeneXus KB was last updated."
    rules: [R65]

  - id: OQ9
    topic: SalaSta values
    desc: "SAU_UNISALA SalaSta is CHAR(1); checkExtendedTable0E28 is empty. What are the valid values ('S'/'N', 'A'/'I', '1'/'2')? Check DB data or domain spec."
    rules: [R81]

  - id: OQ10
    topic: Auth role
    status: RESOLVED 2026-06-21
    resolution: "SAUDE_ADMIN required for POST/PUT/DELETE on /api/unidades. SAUDE_CADASTRO sufficient for sub-resource endpoints and all reads. auth section updated."

  - id: OQ11
    topic: UniCod assignment
    status: RESOLVED 2026-06-21
    resolution: "UniCod is generated (not user-entered). Implemented via DB SEQUENCE seq_sau_uni_cod; sequence created in V4 seeded from MAX(UniCod)."
```

---

## Domain notes

### Constellation

SAU_UNI has the smallest constellation of any XL slice: only `sau_uni.java` (servlet stub), `sau_uni_impl.java` (1.08 MB), and a lookup prompt pair (`hpromptsau_uni.java`/`hpromptsau_uni_impl.java`). No `hwwsau_uni.java`, `hviewsau_uni.java`, section helpers, SDT, or BC. All CRUD modes are handled inline by the impl via GeneXus `Gx_mode` (INS/UPD/DLT). The list/search UI is embedded in `hpromptsau_uni_impl.java` (84 KB).

### Schema status

- **SAU_UNI** is fully defined in V1 (all 55 columns, PK, 2 FKs). V4 fixes 3 CHAR/VARCHAR mismatches and adds 7 missing GeneXus indexes + `UniMunCod → SYS_MUN` FK.
- **SAU_UNI1, SAU_UNI2, SAU_UNI3, SAU_UNISALA** are created in V4.
- SAU_PRO and SAU_PROESP FKs (from all sub-tables) are deferred to the Wave-4 migration.

### Rule count by confidence

| Confidence | Count | Rules |
|---|---|---|
| high | 78 | R1-R50 (exc. R51), R52-R59, R62-R69, R71-R77, R79-R80, R82-R84 |
| medium | 5 | R51, R60, R61, R70, R78 |
| low | 1 | R81 |
| **total** | **84** | |
