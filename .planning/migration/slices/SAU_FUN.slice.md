# SLICE-SPEC — SAU_FUN (Funcionário) · **SYS_PES person-subtype (Wave 0)**

> Employee record — a SYS_PES subtype exactly like SAU_PRO (PK `FunPesCod` = `SYS_PES.PesCod`). Only **5
> columns** are physically on SAU_FUN; name/CPF/address/phones live in SYS_PES (native projection + write-
> back, **no** SYS_PES JPA entity). **Lighter than SAU_PRO:** no CPF/CNS check-digit validation (R12), no
> legacy SAU_LOG audit (R17), only 2 delete-guards. Last meaningful Wave-0 data slice. Extracted 2026-06-30
> (3 mining agents; live-DB-verified). Copy the `profissional/` reference package.

```yaml
slice: SAU_FUN
domain: funcionario                  # package …/funcionario/ ; REST /api/funcionarios
description: Funcionários
wave: 0
complexity: M                        # sau_fun_impl.java ~287 KB
primary_table: SAU_FUN               # PK FunPesCod (bigint = SYS_PES.PesCod), 1148 rows, 5 own cols, ZERO physical FK
constellation:
  transaction: sau_fun.java          # IntegratedSecurityEnabled()=false
  impl: sau_fun_impl.java            # ~287 KB — rule source
  bc: null                           # NOT a BC → live DB authoritative
  workwith: hwwsau_fun_impl.java      # list/search
  view: hviewsau_fun_impl.java        # detail (joins SYS_PES)
  prompt: hpromptsau_fun_impl.java     # /api/funcionarios/lookup
  sections: [hsau_fungeneral_impl.java]
  soundex: inline                    # gx9asafunpesnomsoundex0721 — reuse modern SoundexService (confirm table OQ2)

depends_on:
  - { table: SYS_PES, migrated: false, wave: 0, access: "NATIVE projection + write-back (no entity), like SAU_PRO" }
referenced_by:                       # delete-guards (native EXISTS)
  - { table: SAU_USU,    col: FunPesCod, migrated: false, role: "funcionário linked to a system user (R13)" }
  - { table: SAU_RECESP, col: FunPesCod, migrated: false, role: "controlled-substance prescription (R14, Portaria 344/98 → soft-deactivate)" }

# ── ENTITY: 5 own columns (live-DB authoritative) ──
fields:
  - { gx: FunPesCod,        name: id,               type: long,   pk: true, nullable: false,
      note: "= SYS_PES.PesCod (user-selected existing Pessoa; NOT @GeneratedValue)" }
  - { gx: FunTraFon,        name: telefoneTrabalho, type: string, len: 20, nullable: true, phi: true, note: "work phone (R8 format)" }
  - { gx: FunTraRam,        name: ramal,            type: string, len: 10, nullable: true, jdbc: CHAR, note: "char(10); free text, NO validation (R9)" }
  - { gx: FunPesNomSoundex, name: nomeSoundex,      type: string, len: 50, nullable: true, derived: true, note: "recomputed from SYS_PES name (R3); never client-set; NEVER logged (R18)" }
  - { gx: FunSit,           name: situacao,         type: short,  nullable: true, jdbc: SMALLINT, note: "1=Ativo/2=Inativo; app-default 1 on insert (R5; NO DB default)" }
# Person fields (live on SYS_PES, surfaced via native access; write-back subset like SAU_PRO):
fields_person: [nome(PesNom,phi), cpfCnpj(PesCPFCNPJ char18,phi), telefone(PesFon,phi), celular(PesCel,phi)]
# Full address panel (PesEnd/Cep/Mun/Bai/TipLog + R10/R11 address validations) DEFERRED to the SYS_PES slice
# (same scope decision as SAU_PRO — this slice writes back name/cpf/phones only).

keys: { pk: [FunPesCod], unique: [], fks: [] }   # logical FunPesCod→SYS_PES.PesCod only

endpoints:
  - { method: GET,    path: /api/funcionarios,        auth: SAUDE_CADASTRO, from: hwwsau_fun }
  - { method: GET,    path: /api/funcionarios/{id},   auth: SAUDE_CADASTRO, from: hviewsau_fun }
  - { method: POST,   path: /api/funcionarios,        auth: SAUDE_CADASTRO, note: "person must pre-exist (R1); situacao default 1 (R5)" }
  - { method: PUT,    path: /api/funcionarios/{id},   auth: SAUDE_CADASTRO, note: "writes 5 own + SYS_PES write-back (R2)" }
  - { method: DELETE, path: /api/funcionarios/{id},   auth: SAUDE_CADASTRO, note: "blocked by SAU_USU (R13) / SAU_RECESP (R14)" }
  - { method: GET,    path: /api/funcionarios/lookup, auth: SAUDE_CADASTRO, from: hpromptsau_fun, note: "returns {id, nome} — nome is PII → NOT isAuthenticated (M5 lesson)" }

phi_fields: [nome, cpfCnpj, telefone, celular, telefoneTrabalho]   # LGPD: audit reads/writes; NEVER log (esp. R18)
auth:
  roles_required: [SAUDE_CADASTRO]   # pisauthorized('SAU_FUN', mode) + SYSMAR bypass → coarse role (OQ5)
  source: "sau_fun_impl.java:1243-1267"

parity:
  legacy_base: "auth-gated; behavioral parity vs live DB + mined rules (like SAU_PRO/SAU_IMP)"
  scenarios:
    - "create funcionário for existing person → 201, situacao=1, soundex computed from SYS_PES name"
    - "create for non-existent person → rejected (R1)"
    - "update → writes back name/cpf/phones to SYS_PES (R2)"
    - "invalid work phone / person phone / mobile → rejected (R6/R7/R8); ramal accepts free text (R9)"
    - "NO CPF/CNS check-digit validation (R12 — divergence from SAU_PRO)"
    - "delete blocked when referenced by SAU_USU (R13) or SAU_RECESP (R14); allowed otherwise (R15)"
    - "secrets/PHI never logged (R18)"

status: tested   # backend + testes (19 unit + 12 IT) verdes 2026-06-30; revisão tratada. Suite unit 319/0F/0E.
# Gerado (…/funcionario/): Funcionario (5 cols), FuncionarioRepository (native SYS_PES projection/write-back +
# guards), PersonProjection, FuncionarioService (R1/R2/R3/R5/R6-R9/R12/R13-R15/R16/R17/R18; reusa SoundexService),
# FuncionarioController (/api/funcionarios SAUDE_CADASTRO). V1 baseline + SYS_PES/SAU_RECESP stubs.
# Revisão (migration-reviewer): corrigido — get() agora audita READ (PHI, espelha SAU_PRO); +testes get-audit
# e update-preserva-situacao. R19 (subtype-uniqueness) back-anotada abaixo. PENDENTE: /verify-parity (fixtures
# funcionario) — gate de `verified`, não de `tested` (mesmo padrão de SAU_PRF/SAU_RBAC).
# ÚLTIMA FATIA DE DADOS DO WAVE-0.
status_was: specced
```

## Regras mineradas (citação à linha; confiança)

- **R1** (validation, high) — `FunPesCod` é o código da Pessoa (SYS_PES); a pessoa deve **pré-existir** ("Não existe 'Funcionário'."). Criar funcionário transforma uma Pessoa existente; nunca cria SYS_PES. src `sau_fun_impl.java:1654-1660,3663-3665`. → `FuncionarioServiceTest#rejectsCreateWhenPersonDoesNotExist`.
- **R2** (side-effect, high) — Write-back a SYS_PES (PesNom/CPFCNPJ/CEP/End/EndNum/EndCom/Fon/Cel/NumCns/TipLog/Mun/Bai WHERE PesCod=FunPesCod). INSERT/UPDATE de SAU_FUN tocam só as 5 colunas próprias. **Implementação (como SAU_PRO):** write-back de nome/cpf/fone/cel; endereço completo diferido. src `:2785 (T000726)`. → `FuncionarioServiceTest#writesBackPersonFieldsToSysPes`.
- **R3** (formula, medium) — `FunPesNomSoundex` recomputado do nome SYS_PES via soundex em todo confirm; read-only. Reusar `SoundexService`. src `:2891-2902,133`. → `FuncionarioServiceTest#computesSoundexFromName`.
- **R5** (default, medium) — `FunSit` domínio 1=Ativo/2=Inativo; insert default **1** (app-layer; sem default no DB). src `:271-272,365-369`. → `FuncionarioServiceTest#defaultsSituacaoToAtivoOnInsert`.
- **R6** (validation, high) — `FunPesFon` (telefone pessoa) quando não-vazio: regex `^(\([0-9]{2}\))\s([9])?([0-9]{4})-([0-9]{4})$` senão "Telefone inválido!". src `:1838-1846`. → `#rejectsInvalidPersonPhoneWhenPresent`.
- **R7** (validation, high) — `FunPesCel` mesma regex senão "Celular inválido!". src `:1846-1854`. → `#rejectsInvalidPersonMobileWhenPresent`.
- **R8** (validation, high) — `FunTraFon` (telefone de trabalho, coluna própria) mesma regex senão "Telefone de trabalho inválido!". src `:1862-1868`. → `#rejectsInvalidWorkPhoneWhenPresent`.
- **R9** (validation, high) — `FunTraRam` (ramal) **sem validação** — texto livre. NÃO inventar regra. src `:771,858-860`. → `#ramalIsUnvalidatedFreeText`.
- **R12** (validation, high, **divergência**) — **SAU_FUN NÃO valida dígito/unicidade de CPF/CNS** (sem psau_val_*/psau_ver_*). NÃO portar validação de CPF para o serviço Funcionário (≠ SAU_PRO). src `:absence`. → `#doesNotValidateCpfOrCnsCheckDigits`. (OQ3: centralizar no SYS_PES futuramente?)
- **R13** (validation, high, delete-guard) — Delete bloqueado se referenciado por **SAU_USU** (FunPesCod) → "Usuários do Sistema". src `:2762-2769`. → `#blocksDeleteWhenReferencedByUsuario`.
- **R14** (validation, medium, delete-guard) — Delete bloqueado se referenciado por **SAU_RECESP** (FunPesCod) → controle especial (Portaria 344/98). Preferir soft-deactivate (FunSit=2). src `:2770-2778`. → `#blocksDeleteWhenReferencedByReceituarioControleEspecial`.
- **R15** (validation, high) — Únicos 2 guards; delete permitido quando nenhum referencia. src `:2761-2779`. → `#allowsDeleteWhenUnreferenced`.
- **R16** (authorization, high) — `pisauthorized('SAU_FUN', mode)` + bypass SYSMAR → `@PreAuthorize` SAUDE_CADASTRO. src `:1243-1267`. → `FuncionarioControllerIT` (401/403).
- **R17** (audit, high, **negativo**) — SAU_FUN **não** escreve SAU_LOG no legado. App moderno **deve** auditar via `common/audit` (carry-forward LGPD). → `#auditsCrudViaCommonAudit`.
- **R18** (audit, high, **DEFEITO — NÃO PORTAR**) — legado loga FunPesNomSoundex/FunTraFon/FunSit/FunPesNom no app log em falha de integridade e change-logging. Nome é PHI → app moderno **nunca** loga esses valores. src `:1022-1023,2303-2323`. → `FuncionarioSecurityTest#neverLogsPersonValues`.

- **R19** (validation, high, back-anotada na revisão) — Um único Funcionário por Pessoa: create com `existsById(FunPesCod)` → Conflict "Já existe Funcionário". (Subtype INS acontece uma vez.) src `sau_fun_impl.java` (subtype INS). → `FuncionarioServiceTest#rejectsCreateWhenFuncionarioAlreadyExists`.

(R4 normalize-name, R10/R11 address validations: diferidos com o painel de endereço → fatia SYS_PES.)

## 🔎 Revisão (migration-reviewer) — 2026-06-30
Cobertura de regras OK (R1-R3,R5-R9,R12-R18 implementadas+testadas; R4/R10/R11 diferidas). **Corrigido:** `get()`
não auditava leitura de PHI → agora `audit.record("READ","SAU_FUN",id)` (espelha SAU_PRO; list/lookup seguem o
precedente SAU_PRO de não auditar por-chamada). +testes: `getAuditsPhiRead`, `updatePreservesSituacaoWhenOmitted`.
R19 (1 funcionário/pessoa) back-anotada. Demais: divergências R12 (sem validação de CPF) e R14 (mensagem orienta
inativar) confirmadas corretas; R18 (toString sem soundex) PASS. **Paridade:** sem `parity/funcionario/` ainda →
`/verify-parity SAU_FUN` é o gate de `verified` (não bloqueia `tested`; mesmo padrão das outras fatias da sessão).

## Mapeamento JPA (verificado vivo)
Entidade `Funcionario @Table("SAU_FUN")`: id (Long @Id, não gerado), telefoneTrabalho (varchar20), ramal (char10 `@JdbcTypeCode(CHAR)`), nomeSoundex (varchar50, derivado), situacao (smallint `@JdbcTypeCode(SMALLINT)`). Sem @OneToOne. Native: `PersonProjection` (SELECT SYS_PES … WHERE PesCod=:id; ⚠ coluna sexo é **PesSex** não PesSexo), write-back UPDATE SYS_PES (subset), person-exists EXISTS, delete-guards EXISTS sau_usu/sau_recesp. `toString()` exclui nomeSoundex.

## Plano Flyway (V1 baseline — confirmado vivo)
- SAU_FUN **ausente** do baseline → adicionar `CREATE TABLE SAU_FUN (FunPesCod bigint PK, FunTraFon varchar20, FunTraRam char10, FunPesNomSoundex varchar50, FunSit smallint)`.
- SYS_PES stub (4 cols) **incompleto** → `ALTER ADD COLUMN IF NOT EXISTS` p/ PesCPFCNPJ char18, PesRGIE, PesNumCns char20, PesCEP char8, PesEnd, PesEndNum, PesEndCom, PesFon, PesCel, PesTipLogCod int, PesMunCod int, PesTip smallint, PesNasDat date, PesSex char1.
- SAU_RECESP stub **sem FunPesCod** → `ALTER ADD COLUMN IF NOT EXISTS FunPesCod bigint`. SAU_USU já tem FunPesCod.

## 🔎 OPEN QUESTIONS
- **OQ1 (FunSit default):** app-default 1 (inferido do combo getValidValue) — confirmado nullable sem DB default. Serviço seta 1 no create.
- **OQ2 (soundex):** reusar `SoundexService` (tabela do SAU_PRO) — confirmar que a tabela inline do SAU_FUN é idêntica p/ paridade de busca.
- **OQ3 (CPF/CNS):** legado SAU_FUN não valida — decidir se a validação deve ser centralizada na camada SYS_PES (aplicaria a todas as transações). Por ora: **não validar** (fiel ao legado).
- **OQ4 (R14 regulatório):** confirmar com líder regulatório que funcionário com histórico SAU_RECESP é **soft-deactivated** (FunSit=2), nunca hard-deleted.
- **OQ5 (auth):** mapear pisauthorized('SAU_FUN',mode) → role concreto (usando SAUDE_CADASTRO coarse, como SAU_PRO).
- **OQ6 (endereço):** painel de endereço completo (R4/R10/R11) diferido p/ a fatia SYS_PES.
