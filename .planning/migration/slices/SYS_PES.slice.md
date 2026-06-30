# SLICE-SPEC — SYS_PES (Pessoa) · **person SUPERTYPE / Business Component (Wave 0)**

> The foundational person record underpinning SAU_PRO, SAU_FUN, SAU_PAC, SAU_USU (PK=FK: each subtype's
> `<prefix>PesCod` **is** `SYS_PES.PesCod`). **BC-only — NO standalone CRUD UI**: persons are created/edited
> THROUGH the subtype transactions. 89 columns, 83,105 rows; **the most PHI-dense table in the system**
> (identity docs, address, family, blood type, social name) plus ⚠ a reversible password (`PesSenha`/`PesSenhaKey`).
> Extracted 2026-06-30 (gx-extractor + gx-rule-miner; live-DB-verified). **STATUS: specced — DO NOT
> `/migrate-slice` until the design + security decisions below are signed off.**

```yaml
slice: SYS_PES
domain: pessoa
description: Pessoa
wave: 0
complexity: L            # 89 cols; the value/risk is in the design decisions, not CRUD volume
primary_table: SYS_PES   # PK PesCod (bigint), 89 cols, 83105 rows
bc_only: true            # SUPERTYPE / Business Component — no transaction/hww/hview/hprompt
constellation:
  bc: sys_pes_bc.java            # 446 KB — defaults, FK-existence, name derivations, delete guards, raw persistence
  sdt: SdtSYS_PES.java
  metadata: gxmetadata/sys_pes.json   # authoritative 89-attr field list
subtypes:                # PK=FK supertype; persons written THROUGH these (native projection+write-back)
  - { table: SAU_PRO, migrated: true,  note: "validates CPF/CNS; recomputes PesNomSoundex" }
  - { table: SAU_FUN, migrated: true,  note: "does NOT validate CPF/CNS; address requireds" }
  - { table: SAU_PAC, migrated: false, wave: 7, note: "the big subtype — full person + mãe/social soundex" }
  - { table: SAU_USU, migrated: false, note: "login user is a person (usupescod)" }
depends_on:              # FK lookups (most NOT yet migrated → ordering risk, OQ6)
  - { table: SAU_BAI,    col: PesBaiCod,    migrated: true,  wave: 1 }
  - { table: SAU_TIPLOG, col: PesTipLogCod, migrated: false }
  - { table: SYS_MUN,    col: "PesMunCod, PesNacMunCod", migrated: false }
  - { table: SYS_PAIS,   col: PesPaisCod,   migrated: false, note: "table name inferred" }
  - { table: SYS_ETN,    col: PesEtnCod,    migrated: false, note: "etnia — inferred" }
  - { table: SAU_ORGEMI, col: PesOrgEmiCod, migrated: false, note: "órgão emissor RG" }
  - { table: SAU_CBOR,   col: PesCborCod,   migrated: false }
  - { table: SAU_CEP,    col: PesCEP,       migrated: false }

# ── FIELDS (89 cols; grouped — full list in gxmetadata/sys_pes.json). Live types pinned. ──
fields_groups:
  identity:    [PesCod(pk bigint), PesTip(smallint, default 2=PF), PesNom(varchar50 @!), PesNomFan(varchar50),
                PesCPFCNPJ(char18), PesRGIE(varchar15), PesNumCns(char20), PesNumSoc(char11 NIS)]
  address:     [PesTipLogCod(int→SAU_TIPLOG), PesCEP(char8), PesMunCod(int→SYS_MUN), PesEnd(varchar70 @!),
                PesEndNum(varchar10), PesEndCom(varchar40), PesBaiCod(int→SAU_BAI)]
  contact:     [PesFon(varchar20), PesCel(varchar20), PesFax(varchar20), PesEmail(varchar70), PesHomePage(varchar70), PesObs(varchar300)]
  demographics:[PesNasDat(date), PesSex(char1), PesCor(smallint), PesEtnCod(int→SYS_ETN), PesEstCiv(smallint),
                PesSitFam(smallint), PesFreEsc(char1), PesGraEsc(smallint), PesEscolaridade(smallint),
                PesCborCod(char6→SAU_CBOR), PesTipSan(char3 ⚠SENSITIVE HEALTH)]
  naturalidade:[PesNacTip(smallint), PesPaisCod(int→SYS_PAIS), PesNacDatNat, PesNacNumPor(varchar16),
                PesNacDatEnt, PesNacMunCod(int→SYS_MUN)]
  family:      [PesNomPai(varchar50), PesNomMae(varchar50), PesNomCon(varchar50)]
  civil_docs:  ["certidão pescer* + nova certidão pesnovcer* (CNJ matrícula)", "RG: PesOrgEmiCod/PesIdeEst/PesIdeDat",
                "título: PesEleNum/Zon/Sec", "CTPS: PesTraSer/Num/Est/Dat", "PIS: PesTraPis(char11)"]
  social_name: [PesNomSoc(varchar50), PesUsaNomSoc(boolean), "derived PesNomRegSoc / PesNomRegSocComp (R2/R3)"]   # LGPD Decreto 8.727/2016
  establishment:[PesEstInsMun, PesEstTecResp, PesEstTecResPCPF(char18), PesEstTecRespCR, PesEstProp,
                PesEstPropCPFCNPJ(char18), PesEstRamAtivCod(bigint), "PesEstRamAtivDes(varchar200 NOT NULL — only non-key required)"]
  system:      [PesCadDat(date), PesGerBpa(smallint), PesSauFlag(smallint), PesSasFlag(smallint)]
  soundex:     [PesNomSoundex, PesNomFanSoundex, PesNomMaeSoundex, PesNomSocSoundex (all varchar50, derived)]  # PHI-equivalent (re-identify)
  SECRETS:     [PesSenha(varchar100), PesSenhaKey(varchar100)]   # ⚠ reversible password+key — NEVER expose/log/fixture (OQ2)

keys: { pk: [PesCod], unique: [], fks: "8 lookup FKs (see depends_on); CPF/CNS uniqueness are RULES not constraints" }

endpoints:   # ⚠ PROPOSED ONLY — supertype has NO legacy UI. NO mutation surface (writes go through subtypes).
  - { method: GET, path: /api/pessoas/{id},     desc: "person resolution by PesCod (read; honors social name)", REVIEW: true }
  - { method: GET, path: /api/pessoas/search,   desc: "search by nome/CPF/CNS incl. soundex (subtype person-pick)", REVIEW: true }
  - { method: GET, path: /api/pessoas/lookup,   desc: "autocomplete person-resolution", REVIEW: true }
  # NO POST/PUT/DELETE on /api/pessoas — persons are created/edited via SAU_PRO/SAU_FUN/SAU_PAC.

phi_fields: [nome, nomeFantasia, nomeSocial, cpfCnpj, cns, nis, rgIe, rgUf, tituloEleitor*, ctps*, pisPasep,
             certidao* , dataNascimento, sexo, cor, etniaId, tipoSanguineo, endereco/numero/complemento/cep/bairro/municipio,
             telefone, celular, fax, email, nomePai, nomeMae, nomeConjuge, municipioNascimento, tec/prop CPF,
             nomeSoundex/nomeMaeSoundex/nomeSocialSoundex(PHI-equivalent)]
secrets: [PesSenha, PesSenhaKey]   # reversible password+key — NEVER in DTO/API/OpenAPI/log/fixture
auth: { roles_required: [SAUDE_CADASTRO], notes: "no own module guard (BC); governed by owning subtype. Any read endpoint must be authenticated + AUDITED (PHI)." }

status: tested   # ADDITIVE read slice implemented 2026-06-30 (OQ1 = additive, user-chosen). Suite 327/0F/0E.
# Gerado (…/pessoa/): Pessoa entity (subconjunto read; PesSenha/PesSenhaKey QUARENTENADOS — não mapeados),
# PessoaRepository (search nativo CAST + lookup + findCpfOwners person-wide), PessoaService (get[audit READ
# PHI] + search/lookup honrando nome social R2/R3 + validadores CPF/CNS centralizados aditivos), PessoaController
# (/api/pessoas GET /{id} /search /lookup, SAUDE_CADASTRO, sem mutação). V1 baseline +6 cols SYS_PES.
# SAU_PRO/SAU_FUN INALTERADOS (sem refator @MapsId). Testes: PessoaServiceTest (8) + PessoaControllerIT (7).
# DIFERIDO (sign-off): OQ2 PesSenha (bridge/quarentena), OQ3 wiring dos validadores nos subtipos, OQ6 FKs de
# lookup não migrados. /verify-parity pendente.
status_was: specced
```

## Regras mineradas (citação à linha; confiança)

### Na própria BC (centralizadas hoje)
- **R1** (default, high) — Insert: `PesTip` vazio/0 → default **2** (pessoa física). src `sys_pes_bc.java:361-368`.
- **R2** (derivation, high, **LGPD**) — `PesNomRegSoc` = `PesNomSoc` se `PesUsaNomSoc`, senão `PesNom`. **Nome social deve ser usado onde a pessoa é exibida.** src `sys_pes_bc.java:586-593`.
- **R3** (derivation, high) — `PesNomRegSocComp` = "`<social>` (`<registro>`)" se usa nome social, senão `PesNom`. src `:578-585`.
- **R4-R11** (referential, high) — FK existence quando ≠0: TipLog→SAU_TIPLOG, Mun→SYS_MUN, Bairro→SAU_BAI, Etnia→SYS_ETN, País→SYS_PAIS, MunNasc→SYS_MUN, OrgEmi→SAU_ORGEMI, CBO→SAU_CBOR. src `sys_pes_bc.java:615-727`.
- **R12-R14** (validation, high, delete-guard) — Pessoa não pode ser deletada se referenciada por **SAU_PRO** / **SAU_FUN** / **SAU_PAC** (CannotDeleteReferencedRecord). src `sys_pes_bc.java:1309-1332`.

### Na camada SUBTIPO hoje (candidatas a centralizar — ⚠ mudança de comportamento, OQ3)
- **R15** (derivation, med) — `PesNom` normalizado (psau_limpacaracter) + @! uppercase. src `sau_pro_impl.java:2390-2396`.
- **R16** (validation, med) — CPF/CNPJ check-digit quando não-branco (psau_val_cnpjcpf). src `sau_pro_impl.java:2397-2413`.
- **R17** (validation, med) — **CPF/CNPJ único entre TODAS as pessoas** (psau_ver_cnpjcpf → SELECT FROM SYS_PES). Regra genuinamente de pessoa. src `sau_pro_impl.java:2414-2432`.
- **R18** (validation, med) — CNS check-digit (psau_val_cns). src `sau_pro_impl.java:2454-2461`.
- **R19** (validation, med) — CNS uniqueness **mas hoje escopo SAU_PRO** (FROM SAU_PRO), não SYS_PES. src `sau_pro_impl.java:2462-2480`.
- **R20** (validation, med) — CNS **obrigatório só p/ profissional** (SAU_PRO), não universal. src `sau_pro_impl.java:2447-2453`.
- **R21** (formula, high) — `PesNomSoundex` recomputado de `PesNom` em todo confirm (psau_soundex — já portado). src `sau_pro_impl.java:4264-4275`.
- **R22** (formula, med) — `PesNomFan/Mae/SocSoundex` recomputados nos subtipos que editam esses nomes (prov. SAU_PAC). src `gxmetadata` + `sys_pes_bc.java:5863`.
- **R23** (validation, med) — Requeridos condicionais de endereço (CEP/tiplog/logradouro/número/bairro) + formatos. src `sau_fun_impl.java:1745-1833`.
- **R24** (validation, med) — `PesFon`/`PesCel` formato `(NN) [9]NNNN-NNNN`. src `sau_pro_impl.java:2433-2446`.

## Mapeamento JPA (do schema vivo — 89 cols, PK bigint)
Entidade `Pessoa @Table("SYS_PES")` em `…/pessoa/`. PK `PesCod` Long (não gerado — definido pelo subtipo). char(N) → `@JdbcTypeCode(CHAR)` (pescpfcnpj/pesnumcns/pescep/pessex/pestipsan/pestrapis/…); smallint → `Short @JdbcTypeCode(SMALLINT)`; `pesusanomsoc` Boolean; datas LocalDate. ⚠ `PesSenha`/`PesSenhaKey` → `@JsonIgnore` + **fora de qualquer DTO** (decidir converter/quarentenar — OQ2). Echoes de lookup (PesMunNom/PesBaiNom/…) NÃO são colunas físicas → não mapear (vêm de join). `toString()` exclui todos os PHI/segredos.

## Plano Flyway
SYS_PES já existe (89 cols) em prod → `validate`. O **stub do baseline** (expandido nas fatias SAU_PRO/SAU_FUN com um subconjunto) precisa ser **completado para as 89 colunas** se a entidade `Pessoa` mapeá-las todas (ddl-auto=validate confere só as mapeadas — uma entidade focada pode mapear um subconjunto). Confirmar contra o vivo.

## 🔴 OPEN QUESTIONS — SIGN-OFF NECESSÁRIO ANTES DE /migrate-slice
- **OQ1 (estratégia de entidade — ARQUITETURAL):** entidade `Pessoa` **aditiva** (read/search + validação centralizada, SAU_PRO/SAU_FUN mantêm acesso nativo) **vs** refatorar os subtipos para `@OneToOne/@MapsId`. Acesso nativo já existe nos 2 subtipos migrados → introduzir um agregado `Pessoa` cria **risco de caminho de escrita duplo** na mesma tabela de 83k linhas. **Recomendado: aditivo** (entidade read-only + validadores compartilhados; sem refator dos subtipos). Decidir antes de qualquer código.
- **OQ2 🔴 RESOLVIDA (origem, 2026-06-30; sign-off de segurança ainda pendente):** `PesSenha` é
  **criptografia REVERSÍVEL** — `cadastrarsenha.java:70`: `PesSenha = Encryption.encrypt64(senha, PesSenhaKey)`,
  **mesmo esquema encrypt64 + chave por-linha do SAU_USU**. Definida pelo objeto `cadastrarsenha` e editada
  via `hwwsau_pesf` (work-with de **SAU_PESF** = Pessoa Física / registro de profissional, Wave 4). É um
  **2º caminho de credencial reversível** na pessoa (provável login de portal/autocadastro do profissional).
  **Tratamento idêntico ao SAU_USU:** `@JsonIgnore`, **nunca** em DTO/API/OpenAPI/log/fixture; migrar via
  **bridge p/ bcrypt** (reusar `tools/password-bridge`, adaptado p/ SYS_PES) OU quarentenar; **sign-off de
  segurança antes do cutover**. Ver [[reference-rbac-engine-not-wired]] e a fatia SAU_USU.
- **OQ3 (centralização = mudança de comportamento):** CPF/CNS check-digit+unicidade existem em SAU_PRO mas **não** em SAU_FUN; unicidade de CNS hoje é escopo-SAU_PRO. Centralizar na camada Pessoa muda comportamento (CPF/CNS validado p/ toda pessoa; CNS único person-wide). **Sign-off produto/regulatório.**
- **OQ4 (LGPD nome social):** R2/R3 derivam o nome de exibição mas a BC não FORÇA consumidores a usá-lo. Auditar relatórios/impressões (receita, BPA) p/ garantir uso de `PesNomRegSoc`; impor no mapping do novo app.
- **OQ5 (soundex mãe/social):** recomputo de `PesNomMae/SocSoundex` não localizado em SAU_PRO/SAU_FUN → confirmar em SAU_PAC.
- **OQ6 (ordem de dependências):** 7 dos 8 lookups de FK (TipLog/SYS_MUN/SYS_PAIS/SYS_ETN/OrgEmi/CBOR/CEP) **não são fatias ainda**; nomes `SYS_*` inferidos. Confirmar no KB + sequenciar antes da validação dura de FK.
- **OQ7 (tipo sanguíneo / dado de saúde sensível LGPD Art.11):** `PesTipSan` vive no supertipo; auditar e nunca logar; decidir se sai do contexto de paciente.
- **OQ8 (PesCadDat default-hoje):** não confirmado na BC nem em SAU_PRO/SAU_FUN; verificar no KB qual subtipo seta hoje.
