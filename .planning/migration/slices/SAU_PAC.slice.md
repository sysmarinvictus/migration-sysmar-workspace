# SLICE-SPEC — SAU_PAC (Paciente) · Wave-6 · reference slice · MOST PHI-DENSE

> The patient. SYS_PES person-SUBTYPE (PK PacPesCod = PesCod) — 26 SAU_PAC-owned columns + person data
> written back to SYS_PES (SAU_PRO precedent). Person-level validations are INHERITED from SAU_PESF
> (`pessoa/` — R1-R55). This slice implements the patient DELTA (R1-R20). Live-DB introspected 2026-07-01
> (26 cols, PK pacpescod, 80k rows). **Blocks SAU_RECESP (Portaria 344/98).**

```yaml
slice: SAU_PAC
domain: paciente                       # /api/pacientes
description: Paciente
wave: 6
complexity: XL                         # sau_pac_impl.java 947 KB (rule/screen density; table is only 26 cols)
primary_table: SAU_PAC                 # PK PacPesCod = SYS_PES.PesCod (subtype)
impl: sau_pac_impl.java (+ sau_pac_bc.java cursors)
depends_on:
  - { table: SYS_PES, migrated: true,  note: "person supertype — write path done (SAU_PESF/pessoa)" }
  - { table: SAU_UNI, migrated: true,  note: "PacUniCod + audit-unit FKs" }
  - { table: SAU_BAI/SYS_MUN/SAU_TIPLOG/SAU_ETN/SAU_PAIS/SAU_ORGEMI/SAU_CBOR, migrated: "via SYS_PES" }
referenced_by:
  - { table: SAU_RECESP, on: PacPesCod, role: "DELETE-GUARD (R14, Portaria 344/98) — NOT migrated; raw probe" }
keys: { pk: [PacPesCod] }

# ── SAU_PAC-owned columns (26; live-verified) — map on Paciente entity. Person fields = SYS_PES (Pessoa). ──
fields:
  - { gx: PacPesCod,           name: id,               type: Long, pk: true }           # = PesCod
  - { gx: PacUniCod,           name: unidadeCod,       type: Integer, fk: SAU_UNI, editable: true }   # R5
  - { gx: PacProNum,           name: prontuario,       type: String, jdbc: CHAR, len: 10, editable: true } # R19 free/optional
  - { gx: PacIdNum,            name: numeroIdentificacao, type: Long, editable: true }
  # clinical (PHI)
  - { gx: PacAler,             name: alergia,          type: String, len: 50,  phi: true }
  - { gx: PacCHistDoe,         name: historicoDoencas, type: String, len: 200, phi: true }
  - { gx: PacObi,              name: obito,            type: Integer, jdbc: SMALLINT, phi: true, note: "default 0 (R6)" }
  - { gx: PacInconsciente,     name: inconsciente,     type: Boolean, phi: true }
  - { gx: PacSituacaoRua,      name: situacaoRua,      type: Boolean, phi: true }
  - { gx: PacSurtoPsiquiatrico, name: surtoPsiquiatrico, type: Boolean, phi: true }
  # social
  - { gx: PacRendaFamiliar,    name: rendaFamiliar,    type: Integer, jdbc: SMALLINT, note: "enum 0..8 (R8)" }
  - { gx: PacMeioTransporte,   name: meioTransporte,   type: String, len: 50 }
  - { gx: PacBeneficioSocial,  name: beneficioSocial,  type: Boolean }
  - { gx: PacCNH,              name: cnh,              type: String, len: 11, phi: true }
  # situação / audit
  - { gx: PacSit,              name: situacao,         type: Integer, jdbc: SMALLINT, note: "default 1=Ativo (R7)" }
  - { gx: PacExpEsusErro,      name: exportEsusErro,   type: Integer, jdbc: SMALLINT, editable: false }
  - { gx: PacUsuLogin,         name: usuarioAlteracao,    type: String, len: 20, editable: false }   # R11
  - { gx: PacUsuLoginIns,      name: usuarioInclusao,     type: String, len: 20, editable: false }   # R11
  - { gx: PacUsuDatAlt,        name: dataUltimaAlteracao, type: LocalDateTime, editable: false }      # R11
  - { gx: PacPesUsuCod,        name: usuarioCod,       type: Integer, editable: false }
  - { gx: PacPesUsuLogin,      name: usuarioLogin,     type: String, len: 40, editable: false }
  - { gx: PacPesCadInsUniCod,  name: unidadeInclusaoCod,  type: Integer, fk: SAU_UNI, editable: false } # R9
  - { gx: PacPesCadAltUniCod,  name: unidadeAlteracaoCod, type: Integer, fk: SAU_UNI, editable: false } # R10 = session unidade
  # derived soundex (system-set)
  - { gx: PacPesNomSoundex,    name: nomeSoundex,      type: String, len: 50, editable: false }       # R12
  - { gx: PacPesNomMaeSoundex, name: nomeMaeSoundex,   type: String, len: 50, editable: false }       # R12
  - { gx: PacPesNomSocSoundex, name: nomeSocialSoundex, type: String, len: 50, editable: false }      # R12
# person fields (nome, nomeSocial, nomeMae/Pai, cpfCnpj, cns, rg, dataNascimento, sexo, address, contact,
# etnia, pais, cbo) = SYS_PES — read via projection, written back on save (Pessoa entity reused).

endpoints:                             # SAUDE_CADASTRO; every read+write AUDITED (most PHI-dense slice)
  - { method: GET,    path: /api/pacientes,        note: "search by nome/nomeMae/prontuario/cpf/cns" }
  - { method: GET,    path: /api/pacientes/{id},   note: "detail (audit PHI READ)" }
  - { method: POST,   path: /api/pacientes,        note: "composite create: SYS_PES person + SAU_PAC" }
  - { method: PUT,    path: /api/pacientes/{id},   note: "update SAU_PAC + write-back SYS_PES" }
  - { method: DELETE, path: /api/pacientes/{id},   note: "R14 guard (SAU_RECESP → 409); hard-delete SAU_PAC (keep SYS_PES)" }
  - { method: GET,    path: /api/pacientes/lookup, note: "autocomplete for prescription screens" }
phi_fields: [nome, nomeSocial, cpfCnpj, cns, rg, dataNascimento, sexo, nomeMae, nomePai, cep, endereco,
  numero, complemento, telefone, celular, email, alergia, historicoDoencas, obito, inconsciente,
  situacaoRua, surtoPsiquiatrico, cnh]
auth: { roles_required: [SAUDE_CADASTRO], note: "R16 psau_checarperm(SAU_PAC); clinical-PHI stronger-role = OQ" }
status: tested
status_was: pending
```

## Regras (patient-specific delta; sau_pac_impl.java / sau_pac_bc.java)
- **R1** Pessoa Jurídica (tipoPessoa=2) NÃO pode ser paciente `:4200`. (person tipoPessoa herdado)
- **R2** CPF **ou** CNS obrigatório — salvo se a unidade permite cadastro sem CPF (`SAU_UNI.UniCadCPF`) `:4837,3115`. "Informe o CPF ou CNS!"
- **R3** CNS único **entre PACIENTES** (join SAU_PAC→SYS_PES) `:4784`, psau_ver_pac. (difere da unicidade genérica)
- **R4** CNS válido (CnsValidator) `:4755`. **R (person)** nome-quality + CPF válido/único herdados de SAU_PESF (`PessoaNomeValidator`/`CpfValidator`).
- **R5** PacUniCod opcional; se setado, existe em SAU_UNI `:4283`.
- **R6** PacObi default 0 `:3477`. **R7** PacSit default 1=Ativo `:3483`.
- **R8** PacRendaFamiliar enum {0..8} `:4923`.
- **R9** PacPesCadInsUniCod opcional FK `:4958`. **R10** PacPesCadAltUniCod = unidade da sessão `:7215`.
- **R11** audit cols: PacUsuDatAlt=now, PacUsuLogin=login sessão, PacUsuLoginIns (insert) `:7204`.
- **R12** soundex (nome/mãe/social) via SoundexService `:7218`.
- **R13** se usaNomeSocial → PacPesNomSoc obrigatório ("Informe o Nome Social!") + derivação nome registro/social `:4930`.
- **R14** DELETE-GUARD: bloqueia se existe SAU_RECESP p/ PacPesCod (Portaria 344/98) `bc:7353,2104` → 409. **⚠ SAU_RECESP não migrado — probe raw (`to_regclass` dinâmico).**
- **R15** delete = HARD delete só do SAU_PAC (SYS_PES person mantido) `:6661`.
- **R16** auth psau_checarperm("SAU_PAC") `:3093` → SAUDE_CADASTRO (inferido).
- **R17** auditoria psau_inc_log `:7117` → common/audit.
- **R18** PacDocumentoLGPD = documento mascarado (derivado) `:4205` — LGPD.
- **R19** PacProNum livre/opcional (sem geração/unicidade) `:2475`.
- **R20** flags clínicas/sociais (alergia/histórico/transporte/benefício/CNH/inconsciente/rua/surto) = entrada livre.

## Persistência
- Paciente entity = 26 cols SAU_PAC (PacPesCod raw Long PK, NÃO @MapsId). Person via Pessoa (SYS_PES) — read projection + write-back (SAU_PRO precedent) OU compor com Pessoa entity (write path pronto).
- **V13__sau_pac_paciente.sql**: `ADD COLUMN IF NOT EXISTS` das ~20 colunas SAU_PAC ausentes do stub do baseline + `SAU_RECESP ADD COLUMN IF NOT EXISTS PacPesCod BIGINT` (p/ o guard R14 no Testcontainers; no-op na DB viva).

## Resultado (2026-07-01 — /migrate-slice)
Backend (paciente/: Paciente entity 26 cols + projection + dto + repo + service composto + controller).
Person via `Pessoa` (SYS_PES) — composite create + write-back (SAU_PRO/PESF precedent). Reusa
PessoaNomeValidator (agora todos os métodos public), CnsValidator, CpfValidator, SoundexService,
PessoaRepository.findMaxPesCod/findCpfOwners. Regras R1-R20 (menos as herdadas de pessoa já cobertas).
Testes: Service 16 + IT 10 (Testcontainers — verifica SYS_PES+SAU_PAC criados, guard R14 via SAU_RECESP,
R15 hard-delete mantém pessoa). V13: ADD COLUMN das ~21 cols SAU_PAC + `ALTER PacProNum TYPE CHAR(10)`
(alinha ao live) + `SAU_RECESP.PacPesCod` + `SAU_UNI.UniCadCPF` (todos no-op no live). Sem frontend
(fatia de referência; front do paciente é grande — próximo passo). OpenAPI +6 paths.

## OPEN QUESTIONS — sign-off
- **OQ1 (REGULATÓRIO) — retenção/soft-delete Portaria 344/98:** legado faz HARD delete do SAU_PAC (R15) e só bloqueia se há SAU_RECESP (R14). Confirmar com lead regulatório se o app novo deve trocar hard-delete por soft-deactivation (PacSit=2) p/ preservar trilha LGPD/344, e manter o guard RECESP verbatim.
- **OQ2 (CLÍNICO) — paciente falecido (PacObi=1) → novas prescrições:** SAU_PAC não bloqueia nada por PacObi; se a regra existe, vive no SAU_RECESP. Escalar p/ a fatia SAU_RECESP.
- **OQ3 (LGPD) — PHI clínica:** auditar toda leitura/escrita de alergia/histórico/surto/óbito; nunca em log; máscara CPF/CNS (R18) quando o caller não tem full-view; papel mais forte p/ PHI clínica vs cadastro simples (RBAC não plugado).
- **OQ4 — delete-guards extras:** SAU_PESF_PAC / SAU_PACCNS podem referenciar PacPesCod; o BC do SAU_PAC só checa SAU_RECESP. Confirmar FKs live.
- **OQ5 — sessão-unidade:** R10 (CadAltUniCod = unidade da sessão) precisa de unidade na sessão; o modelo de auth atual (JWT) não carrega unidade. Aceitar do request/deixar null por ora (OQ).
- **OQ6 — parity:** `/verify-parity SAU_PAC` (comparar SAU_PAC + write-back SYS_PES no golden-master).

## REVIEW (2026-07-01, migration-reviewer) — core PASS; ações tomadas
Reviewer rodou os testes (Service 16→**19**, IT 10, 0 falhas). PHI/audit/security/schema = PASS.
Ajustes feitos após o review:
- **Testes de `update()` adicionados** (3): mantém CNS próprio (selfId exclui R3), rejeita CNS de outro
  paciente, not-found. Era a lacuna real na fatia mais sensível.
- **R9/R10 (colunas unidade-audit) e R18 (máscara LGPD) marcadas explicitamente DIFERIDAS no código**
  (comentário honesto — antes o `// R10/R11` implicava feito). R9/R10 = precisam de unidade na sessão
  (OQ5); R18 = máscara CPF/CNS só faz sentido p/ caller sem full-view (hoje só SAUDE_CADASTRO acessa, OQ3).
- **FLAGs resolvidos com introspecção live:** (a) `PacProNum` no live É `character(10)` → o `ALTER ... TYPE
  CHAR(10)` do V13 é metadata-only (sem rewrite/lock na tabela de 80k). (b) O live **NÃO** tem coluna
  `SAU_PAC.PacPesNom` (só o stub do baseline a criou como artefato) → não há cópia denormalizada p/ manter.
  (c) `SAU_UNI.UniCadCPF` já existe no baseline (V1:266) → o ADD do V13 é no-op redundante (inofensivo).
- **Permanecem como gates de cutover (OQ):** OQ1 (soft vs hard delete Portaria 344 — sign-off), OQ4 (guards
  extras SAU_PACCNS/SAU_PESF_PAC — confirmar FKs live), OQ6 (parity). PK MAX+1 tem janela de corrida
  (igual ao legado) — revisitar com sequence.
