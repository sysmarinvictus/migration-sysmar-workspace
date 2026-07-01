# SLICE-SPEC — SAU_PESF (Pessoa Física) · Wave-4 · person-supertype WRITE path

> **⚠ HIGHEST-STAKES SLICE SO FAR — REVIEW BEFORE `/migrate-slice`.** SAU_PESF is *not its own table*:
> it is the INSERT/UPDATE/DELETE transaction over **SYS_PES**, the person supertype that SAU_PRO / SAU_FUN /
> SAU_PAC all extend (PK=FK). This slice **write-enables the existing additive read model** in
> `receituario-modern/.../pessoa/` — it adds no new entity/table. PHI-heavy (~40 PHI columns), touches
> credential columns (PesSenha/PesSenhaKey — quarantined), 59 mined rules, and the legacy After-Trn
> auto-provisions subtypes. Live-DB introspected 2026-06-30 (86 SYS_PES columns, PK pescod, no physical FKs).

```yaml
slice: SAU_PESF
domain: pessoa                     # EXTENDS the existing pessoa/ package — do NOT create pessoa-fisica/
description: Pessoa Física
wave: 4
complexity: L                      # sau_pesf_impl.java ~613 KB
primary_table: SYS_PES             # INSERT/UPDATE/DELETE targets — sau_pesf_impl.java:9502-9504, key PesCod
is_new_table: false                # write-enables an already-migrated entity (Pessoa @Table(SYS_PES))
impl: sau_pesf_impl.java           # rule source (+ psau_pesf_fun/pro/pac, psau_val_*, psau_soundex, psau_valcep)
depends_on:
  - { table: SYS_PES,    migrated: partial, note: "read model exists (pessoa/); this adds the write path" }
  - { table: SAU_TIPLOG, migrated: true,  col: PesTipLogCod,  role: FK-lookup }
  - { table: SAU_BAI,    migrated: true,  col: PesBaiCod,     role: FK-lookup }
  - { table: SYS_MUN,    migrated: "?",   col: "PesMunCod, PesNacMunCod", role: FK-lookup }
  - { table: SAU_ETN,    migrated: "?",   col: PesEtnCod,     role: FK-lookup }
  - { table: SAU_PAIS,   migrated: "?",   col: PesPaisCod,    role: FK-lookup }
  - { table: SAU_CBOR,   migrated: true,  col: PesCborCod,    role: FK-lookup (CHAR6) }
  - { table: SAU_ORGEMI, migrated: "?",   col: PesOrgEmiCod,  role: FK-lookup }
referenced_by:            # delete-guards (raw table probes) + subtype auto-provisioning targets
  - { table: SAU_PRO, migrated: true,  role: "delete-guard (R53) + provisioning target (R57)" }
  - { table: SAU_FUN, migrated: tested, role: "delete-guard (R54) + provisioning target (R56)" }
  - { table: SAU_PAC, migrated: false, role: "delete-guard (R55) + provisioning target (R58) — Wave 6, NOT migrated" }

keys: { pk: [PesCod], physical_fks: none }   # GeneXus declares no physical FKs (live-confirmed)

# ── ENTITY: delta on pessoa/domain/Pessoa.java (@Entity @Table(SYS_PES)). Already-mapped read subset
#    (id, nome, nomeSocial, usaNomeSocial, nomeSoundex, cpfCnpj[CHAR18], cns[CHAR20], dataNascimento,
#    sexo[CHAR1], telefone, celular, email) is reused. New WRITE columns below — all nullable, none FK.
#    Types LIVE-VERIFIED. CHAR(n) → @JdbcTypeCode(SqlTypes.CHAR) + trim-on-read getter (PG blank-pads). ──
fields_delta:
  - { gx: PesTip,        name: tipoPessoa,        type: Integer, jdbc: SMALLINT, note: "default 2=física on insert (R48)" }
  - { gx: PesNomPai,     name: nomePai,           type: String, len: 50, phi: true }
  - { gx: PesNomMae,     name: nomeMae,           type: String, len: 50, phi: true }
  - { gx: PesNomCon,     name: nomeConjuge,       type: String, len: 50, phi: true }
  - { gx: PesNomMaeSoundex, name: nomeMaeSoundex, type: String, len: 50, readonly: true, note: "derived (R52)" }
  - { gx: PesNomSocSoundex, name: nomeSocialSoundex, type: String, len: 50, readonly: true }
  - { gx: PesRGIE,       name: rgIe,              type: String, len: 15, phi: true }
  - { gx: PesOrgEmiCod,  name: orgaoEmissorCod,   type: Integer, fk: SAU_ORGEMI }
  - { gx: PesIdeEst,     name: rgUf,              type: String, jdbc: CHAR, len: 2, phi: true }
  - { gx: PesIdeDat,     name: rgDataEmissao,     type: LocalDate }
  - { gx: PesCor,        name: corCod,            type: Integer, jdbc: SMALLINT, phi: true, note: "5=indígena → etnia required (R24)" }
  - { gx: PesEstCiv,     name: estadoCivilCod,    type: Integer, jdbc: SMALLINT, phi: true }
  - { gx: PesSitFam,     name: situacaoFamiliarCod, type: Integer, jdbc: SMALLINT, note: "default 0 (R50)" }
  - { gx: PesEtnCod,     name: etniaCod,          type: Integer, jdbc: SMALLINT, fk: SAU_ETN, phi: true }
  - { gx: PesTipSan,     name: tipoSanguineo,     type: String, jdbc: CHAR, len: 3, phi: true }
  - { gx: PesNacTip,     name: nacionalidadeTipo, type: Integer, jdbc: SMALLINT, phi: true, note: "1=bra 2=estr 3=nat (R26-R33)" }
  - { gx: PesPaisCod,    name: paisCod,           type: Integer, jdbc: SMALLINT, fk: SAU_PAIS, phi: true }
  - { gx: PesNacMunCod,  name: municipioNascCod,  type: Integer, fk: SYS_MUN, phi: true }
  - { gx: PesNacDatNat,  name: dataNaturalizacao, type: LocalDate }
  - { gx: PesNacNumPor,  name: numeroPortaria,    type: String, len: 16 }
  - { gx: PesNacDatEnt,  name: dataEntradaPais,   type: LocalDate }
  - { gx: PesCEP,        name: cep,               type: String, jdbc: CHAR, len: 8, phi: true }
  - { gx: PesTipLogCod,  name: tipoLogradouroCod, type: Integer, fk: SAU_TIPLOG }
  - { gx: PesEnd,        name: endereco,          type: String, len: 70, phi: true }
  - { gx: PesEndNum,     name: enderecoNumero,    type: String, len: 10, phi: true }
  - { gx: PesEndCom,     name: enderecoComplemento, type: String, len: 40, phi: true }
  - { gx: PesBaiCod,     name: bairroCod,         type: Integer, fk: SAU_BAI, phi: true }
  - { gx: PesMunCod,     name: municipioCod,      type: Integer, fk: SYS_MUN, phi: true }
  - { gx: PesFax,        name: fax,               type: String, len: 20 }
  - { gx: PesHomePage,   name: homePage,          type: String, len: 70 }
  # civil-registry / certidão block (PHI — identity docs)
  - { gx: PesCerCiv,     name: certidaoCivilTipo, type: Integer, jdbc: SMALLINT }
  - { gx: PesCerNum,     name: certidaoNumero,    type: String, len: 8, phi: true }
  - { gx: PesCerLiv,     name: certidaoLivro,     type: String, len: 8 }
  - { gx: PesCerFol,     name: certidaoFolha,     type: String, len: 4 }
  - { gx: PesCerDat,     name: certidaoData,      type: LocalDate }
  - { gx: PesCerCar,     name: certidaoCartorio,  type: String, len: 60 }
  - { gx: "PesNovCer*",  name: "novaCertidao* (Ser/Ace/RegCiv/Ano/TipLiv[CHAR1]/Liv/Fol/Ter/Dv)", type: String, note: "novo modelo de certidão — 10 cols, see V10 DDL" }
  # trabalho / eleitor / social (PHI)
  - { gx: "PesTra*",     name: "ctps* (Serie/Numero/Uf[CHAR2]/Data/pisPasep[CHAR11])", type: mixed, phi: true }
  - { gx: "PesEle*",     name: "tituloEleitor* (Numero/Zona/Secao)", type: String, phi: true }
  - { gx: PesNumSoc,     name: nis,               type: String, jdbc: CHAR, len: 11, phi: true }
  - { gx: PesFreEsc,     name: frequentaEscola,   type: String, jdbc: CHAR, len: 1 }
  - { gx: PesGraEsc,     name: grauEscolaridade,  type: Integer, jdbc: SMALLINT }
  - { gx: PesEscolaridade, name: escolaridade,    type: Integer, jdbc: SMALLINT, note: "LIVE-ONLY col; parity confirms which of GraEsc/Escolaridade is written (§disc-2)" }
  - { gx: PesCborCod,    name: cboCod,            type: String, jdbc: CHAR, len: 6, fk: SAU_CBOR }
  - { gx: PesCadDat,     name: dataCadastro,      type: LocalDate, note: "default today on insert (R49)" }
  - { gx: PesObs,        name: observacao,        type: String, len: 300, phi: true }
  - { gx: PesGerBpa,     name: gerarBpa,          type: Integer, jdbc: SMALLINT }

fields_quarantined:    # NOT mapped — reversible person credential; security sign-off pending (OQ-SEC)
  - { gx: PesSenha,    note: "varchar100; legacy INSERT writes '' — modern write path must NOT set it" }
  - { gx: PesSenhaKey, note: "varchar100 key/salt" }

fields_deferred:       # establishment/PJ block — map ONLY if PJ cadastro is in scope (default: DEFER)
  - "PesNomFan, PesNomFanSoundex, PesEstInsMun, PesEstTecResp, PesEstTecRespCPF[CHAR18], PesEstTecRespCR, PesEstProp, PesEstPropCPFCNPJ[CHAR18], PesEstRamAtivCod[BIGINT]"
  - "⚠ PesEstRamAtivDes is in gxmetadata but ABSENT from the live table — do NOT map/DDL (§disc-1)"

endpoints:   # ATTACH to the existing /api/pessoas controller — do NOT create a parallel surface
  - { method: GET,    path: /api/pessoas,        note: "list/search — already exists (read model)" }
  - { method: GET,    path: /api/pessoas/{id},   note: "detail — already exists (read model)" }
  - { method: POST,   path: /api/pessoas,        note: "NEW: create cadastro (R1-R50 validations, R48/R49/R50 defaults, R51/R52 soundex)" }
  - { method: PUT,    path: /api/pessoas/{id},   note: "NEW: update cadastro" }
  - { method: DELETE, path: /api/pessoas/{id},   note: "NEW: delete; blocked by SAU_PRO/SAU_FUN/SAU_PAC (R53/R54/R55)" }
phi_fields: [nome, nomeSocial, nomePai, nomeMae, nomeConjuge, cpfCnpj, cns, rgIe, rgUf, dataNascimento,
  sexo, corCod, estadoCivilCod, etniaCod, tipoSanguineo, nacionalidadeTipo, paisCod, municipioNascCod,
  cep, endereco, enderecoNumero, enderecoComplemento, bairroCod, municipioCod, telefone, celular, fax,
  email, certidaoNumero, certidaoData, ctpsNumero, ctpsSerie, pisPasep, tituloEleitorNumero, nis, observacao]
auth: { roles_required: [SAUDE_CADASTRO], enforced_via: "pisauthorized(\"SAU_PESF\") impl:2313-2331 (INFERENCE — confirm program/permission)" }
status: specced
status_was: pending
```

## Regras mineradas (citação `sau_pesf_impl.java:linha` salvo indicação) — 59 regras
**Nome (R1–R9)** — R1 obrigatório `:3185`; R2 ≥3 chars `:3199`; R3 exige sobrenome/espaço `:3206`; R4 sem espaço duplo `:3213`; R5 só letras/espaço/apóstrofo `:3220`; R6 não 2 termos de 1 char `:3227`; R7 não 2 termos de 2 chars `:3234`; R8 sem caractere único (exceto E/Y) `:3241` *(med — portar regex verbatim)*; R9 normaliza PesNom/End/EndCom/NomPai/NomMae/NomCon via `psau_limpacaracter` `:3192,3287,3308,3617,3673,3680` *(med)*.
**Endereço (R10–R16)** — R10 CEP obrigatório `:3248`; R11 CEP válido p/ município via `psau_valcep` `:3371` *(med)*; R12 tipoLogradouro obrigatório+FK SAU_TIPLOG `:3255,3273`; R13 logradouro obrigatório `:3280`; R14 número obrigatório + dígitos ou 'SN' `:3294`; R15 bairro obrigatório+FK SAU_BAI `:3315`; R16 município obrigatório+FK SYS_MUN `:3338`.
**Contato (R17–R18)** — R17 telefone formato `(NN) [9]NNNN-NNNN` `:3385`; R18 celular idem `:3392`.
**Nascimento (R19–R21)** — R19 dataNasc obrigatória `:3419`; R20 não futura `:3770`; R21 idade ≤130 anos `:3777`.
**Sexo/cor/etnia (R22–R25)** — R22 sexo obrigatório `:3440`; R23 cor/raça obrigatória `:3447`; R24 cor==5 (indígena) ⇒ etnia obrigatória `:3454`; R25 etnia FK SAU_ETN `:3461`.
**Nacionalidade (R26–R38)** — R26 obrigatória `:3477`; R27 país obrigatório salvo nacTip==3 `:3484`; R28 estrangeiro⇒país≠Brasil(10) `:3491` *(med — const mágica)*; R29 brasileiro⇒país==Brasil(10) `:3498` *(med)*; R30 estrangeiro⇒dataEntrada obrig. `:3505`; R31 naturalizado⇒dataNat obrig. `:3512`; R32 naturalizado⇒portaria obrig. `:3519`; R33 brasileiro⇒munNasc obrig.+FK `:3526,3549`; R34 país FK SAU_PAIS `:3533`; R35 dataNat ≥ dataNasc `:3426`; R36 dataEntrada ≥ dataNasc `:3433`; R37 dataNat não futura `:3786`; R38 dataEntrada não futura `:3793`.
**Filiação/social (R39–R41)** — R39 nomePai (mesmas regras de nome, se preenchido) `:3568`; R40 nomeMae idem `:3624`; R41 nomeSocial sem espaço duplo/só letras `:3747`.
**CNS (R42–R43)** — R42 CNS obrigatório `:3687`; R43 valida via `psau_val_cns` (1=DV inválido,2=não numérico,3=tam≠15) `:3164,3694` *(med — reusar `common/validation/CnsValidator`)*.
**CPF/CNPJ (R44–R45)** — R44 **OPCIONAL**, mas se presente válido via `psau_val_cnpjcpf` `:3157,3702` *(med — reusar `CpfValidator`)*; R45 CPF único em SYS_PES (exclui PesCod atual) via `psau_ver_cnpjcpf` → "utilizado pelo cadastro <PesCod>" `:3712`.
**Outras FKs (R46–R47)** — R46 CBOR FK SAU_CBOR `:3731`; R47 órgão emissor FK SAU_ORGEMI `:3757`.
**Defaults insert (R48–R50)** — R48 PesTip=1/2 (física) `:2839`; R49 PesCadDat=data servidor `:2846`; R50 PesSitFam=0 `:575` *(med)*.
**Soundex (R51–R52)** — R51 PesNomSoundex ← `psau_soundex(PesNom)` `:6425`; R52 PesNomMaeSoundex idem `:6448` *(med — **reusar `profissional/service/SoundexService`**; parity de busca depende do algoritmo exato)*.
**Delete-guards (R53–R55, high)** — R53 bloqueia se SAU_PRO referencia (cursor `SELECT ProPesCod FROM SAU_PRO WHERE ProPesCod=?`) `:5163,9513`; R54 SAU_FUN idem `:5171,9514`; R55 SAU_PAC idem `:5179,9515` *(SAU_PAC não migrado — query raw)*.
**Side-effects / provisionamento de subtipo (R56–R58) — ⚠ decisão de workflow (OQ1)** — R56 tipoCadastro==1 ⇒ auto-INSERT SAU_FUN (FunSit=1) via `psau_pesf_fun` `:2385`; R57 tipoCadastro==2 ⇒ auto-INSERT SAU_PRO (ProSit=1, cert/senha vazios) via `psau_pesf_pro` `:2407`; R58 provisionamento SAU_PAC (PacSit=1,PacObi=0) via `psau_pesf_pac` *(low — caminho do caller não confirmado)*.
**Autorização (R59)** — `pisauthorized("SAU_PESF", Gx_mode)` por modo INS/UPD/DLT `:2313` *(med — RBAC SAU_RBAC construído mas não plugado)*.
**Auditoria (LGPD)** — o legado NÃO tem SAU_LOG nesta trn; no app novo TODA leitura/escrita de PHI em SYS_PES deve passar por `common/audit` (ver open_questions).

## Persistência (schema-mapper — LIVE-DB introspectado 2026-06-30)
- Entidade: **estender `pessoa/domain/Pessoa.java`** com as colunas `fields_delta` (tipos verificados no PG vivo). Reusar padrão CHAR-trim já existente (cpfCnpj/cns/sexo).
- FKs: **sem `@ManyToOne`** — todas as refs são colunas-código GeneXus (o PG vivo não tem FK física); mapear como `@Column` id + lookup separado, como SAU_PRO/SAU_FUN.
- Flyway: **`V10__sau_pesf_pessoa_write_columns.sql`** — `ADD COLUMN IF NOT EXISTS` idempotente p/ as ~62 colunas novas (DDL completo no output do schema-mapper; DB real já as tem → no-op). Já presentes no baseline (NÃO readicionar): PesCod, PesNom, PesNomSoundex, PesBaiCod, PesCPFCNPJ(CHAR18), PesFon, PesCel, PesNomSoc, PesUsaNomSoc(BOOL), PesNumCns(CHAR20), PesNasDat, PesSex(CHAR1), PesEmail.

## Discrepâncias p/ a parity pegar
1. **PesEstRamAtivDes** — em gxmetadata, AUSENTE no DB vivo → não mapear/DDL. (§disc-1)
2. **PesEscolaridade vs PesGraEsc** — DB vivo tem AMBAS; gxmetadata só GraEsc. Parity confirma qual SAU_PESF escreve. (§disc-2)
3. **pessasflag / pessauflag** (smallint, live-only) — não mapear salvo se a trn escreve; parity confirma intactas.
4. **PesTip default 2** — nullable no DB; o service deve default `tipoPessoa=2`, não depender do DB.
5. **PesUsaNomSoc = boolean nativo** no PG — mapping `Boolean` atual correto.
6. **Soundex** (Nom/NomMae/NomSoc/NomFan) — o legado grava; se parity comparar bytes da linha, o write novo precisa reproduzir o soundex ou diverge.
7. **CHAR(n) blank-padded** — todo `@JdbcTypeCode(CHAR)` precisa getter trim; comparações de parity trimam os dois lados.

## OPEN QUESTIONS — resolver ANTES de `/migrate-slice` (sign-off humano)
- **OQ1 — Escopo do provisionamento de subtipo (R56/R57/R58). DECISÃO DE PRODUTO.** O legado, ao salvar uma Pessoa Física, cria automaticamente SAU_FUN/SAU_PRO/SAU_PAC conforme `tipoCadastro` e redireciona. **Recomendação:** SAU_PESF v1 = **cadastro puro de pessoa** (POST/PUT/DELETE em SYS_PES); o provisionamento de subtipo permanece nos fluxos existentes de SAU_PRO/SAU_FUN. Motivos: (a) **SAU_PAC é Wave 6 e não existe** → não dá para provisionar paciente; (b) cross-entity writes numa fatia de supertipo aumentam muito o raio de teste/risco; (c) mantém a fatia aditiva e reversível. **Alternativa:** implementar provisionamento SAU_FUN/SAU_PRO agora (sem paciente) via um campo `tipoCadastro` no request. → **precisa da escolha do usuário.**
- **OQ2 — Credenciais PesSenha/PesSenhaKey.** Mantidas **quarentenadas** (não mapeadas). O write novo insere linha SYS_PES SEM tocar nessas colunas (legado grava ''). Confirmar que deixá-las NULL em linhas novas é aceitável (vs. legado ''); e o sign-off de segurança da senha reversível continua pendente (compartilhado com SYS_PES OQ2). Ver [[reference_sys_pes_supertype]].
- **OQ3 — Auditoria LGPD.** O legado não audita; o app novo DEVE auditar toda leitura/escrita de PHI em SYS_PES (`common/audit`). Confirmar o volume (READ em GET/{id} e search) para não gerar ruído excessivo.
- **OQ4 — RBAC / permissão.** `SAUDE_CADASTRO` é inferência; confirmar o programa/permissão real de SAU_PESF (KB / AcessaModulo.xml). RBAC por-programa (SAU_RBAC) existe mas não está plugado — ver [[reference_rbac_engine_not_wired]].
- **OQ5 — Algoritmos de identificador.** CPF/CNPJ (`psau_val_cnpjcpf`), CNS (`psau_val_cns`) e soundex (`psau_soundex`) já têm equivalentes no app (`CpfValidator`/`CnsValidator`/`SoundexService`). Confirmar equivalência exata dos DVs/algoritmo p/ parity antes de confiar.
- **OQ6 — Colisão com o read model.** O POST/PUT em `/api/pessoas` passa a ser escrita num controller hoje read-only (SAUDE_CADASTRO). O read atual expõe subconjunto; garantir que o response de escrita não vaze PHI além do já exposto e que a validação (Zod/RHF no front) cubra as 59 regras.
- **OQ7 — SYS_MUN/SAU_ETN/SAU_PAIS/SAU_ORGEMI** status de migração não confirmado; são lookups por-id raw (não bloqueiam), mas confirmar antes do cutover.

## Notas
- Nada de código ainda — só este spec (gx-extract para antes da geração). Reference slice de pessoa-subtype: `profissional/` (SAU_PRO, mesmo padrão CHAR-trim + SoundexService + CertificadoSenhaCryptoConverter).
- Fontes legadas (READ-ONLY, ISO-8859+ugrep → `LC_ALL=C` + `grep -a`): `sau_pesf_impl.java` (:9502-9504 DML, :5163-5186 guards, :2385-2431 After-Trn, :2313 auth), `psau_pesf_fun/pro/pac.java`, `psau_val_cnpjcpf/cns.java`, `psau_ver_cnpjcpf.java`, `psau_soundex.java`, `psau_limpacaracter.java`, `psau_valcep.java`.
