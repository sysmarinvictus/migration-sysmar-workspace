# SLICE-SPEC — SAU_PAR (Parâmetros do sistema) · Wave-5 · config singleton

> Per-empresa **singleton** config row (PK `ParEmpCod`), edited by two legacy transactions: `sau_par_ger`
> (general) + `sau_par_amb` (ambulatorial). Live table has **232 columns**; this slice maps a FOCUSED,
> receituário-relevant subset (user-chosen scope). Live-DB introspected 2026-07-01.

```yaml
slice: SAU_PAR
domain: parametro                      # /api/parametros
description: "Parâmetros do sistema (SAU_PAR singleton) — general + ambulatorial views"
wave: 5
complexity: L                          # sau_par_amb ~582 KB; sau_par_ger ~138 KB (one table, two trns)
primary_table: SAU_PAR                 # singleton: 1 row, PK ParEmpCod (empresa/tenant)
impl: [sau_par_ger_impl.java, sau_par_amb_impl.java]
depends_on: []                         # self-contained config (FK lookups in the deferred blocks only)
keys: { pk: [ParEmpCod] }
tenant: "ParEmpCod = app tenant (reused from AuditProperties.empresaCodigo; test=1, prod default 411420)"

# ── MAPPED subset (19 cols; live-verified types). smallint→Integer+@JdbcTypeCode(SMALLINT);
#    native boolean→Boolean; char(n)→trimmed getter. ──
fields:
  - { gx: ParEmpCod, name: empresaCod, type: Integer, pk: true }
  # general — prescription validity
  - { gx: ParValidadeReceita,        name: validadeReceita,                      type: Boolean }
  - { gx: ParValidadeReceitaSimples, name: validadeReceitaSimplesDias,           type: Integer, jdbc: SMALLINT }
  - { gx: ParValidadeReceitaUsoCon,  name: validadeReceitaUsoContinuoDias,       type: Integer, jdbc: SMALLINT }
  - { gx: ParValidadeReceitaConEsp,  name: validadeReceitaControleEspecialDias,  type: Integer, jdbc: SMALLINT }
  # general — user/security (validated)
  - { gx: ParInaUsuDias, name: inatividadeUsuarioDias, type: Integer, jdbc: SMALLINT, note: "R1 req + R2 ≤180" }
  - { gx: ParSenUsuDias, name: senhaUsuarioDias,       type: Integer, jdbc: SMALLINT, note: "R1 req + R2 ≤180" }
  # general — secretariat report header
  - { gx: ParSecr,      name: secretaria,        type: String, len: 50 }
  - { gx: ParSecrEnd,   name: secretariaEndereco, type: String, len: 50 }
  - { gx: ParSecrCep,   name: secretariaCep,     type: String, jdbc: CHAR, len: 8 }
  - { gx: ParSecrFone1, name: secretariaFone1,   type: String, jdbc: CHAR, len: 20 }
  - { gx: ParSecrFone2, name: secretariaFone2,   type: String, jdbc: CHAR, len: 20 }
  - { gx: ParSecrEmail, name: secretariaEmail,   type: String, len: 70 }
  # general — policy flags
  - { gx: ParCadSemCns,    name: cadastroSemCns,  type: Boolean }
  - { gx: ParRecComprador, name: reciboComprador, type: Boolean }
  # ambulatorial — policy flags
  - { gx: ParExigeCid10Atestado,  name: exigeCid10Atestado,   type: Boolean }
  - { gx: ParEstornarAtendimento, name: estornarAtendimento,  type: Boolean }
  - { gx: ParImpRiscoMaterno,     name: imprimeRiscoMaterno,  type: Integer, jdbc: SMALLINT }
  - { gx: ParAteHis,              name: atendimentoHistorico, type: Integer, jdbc: SMALLINT }

endpoints:                             # config administration → SAUDE_ADMIN; singleton (no create/delete)
  - { method: GET, path: /api/parametros,               note: "read the mapped subset" }
  - { method: PUT, path: /api/parametros/geral,          note: "update general (sau_par_ger) — R1/R2 on dias" }
  - { method: PUT, path: /api/parametros/ambulatorial,   note: "update ambulatorial policy flags (sau_par_amb)" }
phi_fields: []                         # system config — no patient data
auth: { roles_required: [SAUDE_ADMIN] }
status: tested
status_was: pending
```

## Regras (sau_par_ger_impl.java)
- **R1** — `ParInaUsuDias` obrigatório (0 → "Informe a quantidade de dias!") `:1125`; `ParSenUsuDias` idem `:1139`.
- **R2** — `ParInaUsuDias` ≤ 180 ("Quantidade de dias não pode ser superior a 180 dias!") `:1132`; `ParSenUsuDias` ≤ 180 `:1146`.
- Auditoria em cada update (SAU_PAR_GER / SAU_PAR_AMB).
- **par_amb**: os required "Informe o Procedimento!"/"Informe o Tipo de Atendimento" (`:4628/:4596`) pertencem ao **bloco de config de procedimento SIA** (`ParPcd*/ParAtePcd*/ParProSocPcd*`) — DIFERIDO (ver OQ2). A view ambulatorial deste slice cobre as flags de política (Cid10/estorno/risco-materno/histórico), sem regras obrigatórias.

## Persistência (LIVE-introspectado 2026-07-01)
- SAU_PAR = singleton, 232 colunas, PK ParEmpCod, 1 linha. **Não estava no baseline** → `V12__sau_par_singleton.sql` cria a tabela com o SUBSET mapeado (19 cols); na DB viva `CREATE TABLE IF NOT EXISTS` é no-op. `ddl-auto=validate` só checa as 19 mapeadas → passa contra as 232 da DB viva.
- Tenant: a linha é por-empresa; o código vem de `AuditProperties.empresaCodigo` (test=1). GET/PUT operam na linha do tenant configurado; sem create/delete (singleton pré-existente).

## Resultado (2026-07-01 — /migrate-slice)
Backend (parametro/: Parametro entity singleton + repo + 2 update DTOs + service + controller). Testes:
Service 8 + IT 7 (Testcontainers). OpenAPI +3 paths. Sem frontend (config admin; sem tela de admin no
front ainda — como SAU_USUUNI/RBAC).

## Open questions
- **OQ1 — escopo:** só o subset de alto valor foi mapeado (decisão do usuário). ~215 colunas de config ficam não-mapeadas (seguro sob `validate`). Ampliar por demanda.
- **OQ2 — bloco de procedimento SIA de par_amb DIFERIDO** (`ParPcd*/ParAtePcd*/ParProSocPcd*` + suas regras "Informe o Procedimento/Tipo de Atendimento", cursores de código de procedimento SIA/BPA). Sub-slice futuro se necessário.
- **OQ3 — validade de receita:** `ParValidadeReceita` (bool) + os 3 dias (simples/uso-contínuo/controle-especial) são centrais ao receituário — quando SAU_RECESP/receita for migrado, consumir estes parâmetros para a regra de validade.
- **OQ4 — tenant:** reusa `AuditProperties.empresaCodigo`. Se o app virar multi-empresa, extrair um `TenantContext` dedicado.
- **OQ5 — parity:** `/verify-parity SAU_PAR` (comparar a linha singleton geral + ambulatorial no golden-master).
- **OQ6 — null vs 0 / full-replace:** `validateDias` rejeita `null` E `0`; o legado rejeita só `0`. Os PUT são full-replace (o cliente envia o objeto completo), então omitir um campo → null → erro/limpeza. Aceitável p/ um editor de config (form envia tudo); confirmar na parity. Baixo risco (admin-only, os 2 dias sempre no form).

## Confirmação de tipos (LIVE introspection 2026-07-01 — resolve o FLAG de type-provenance do review)
Todas as 19 colunas foram introspectadas na DB viva `saude-mandaguari` via `information_schema.columns`:
`parexigecid10atestado`=**boolean**, `parreccomprador`=**boolean**, `parcadsemcns`=**boolean**,
`parvalidadereceita`=**boolean**, `parestornaratendimento`=**boolean** → `Boolean`;
`parimpriscomaterno`=**smallint**, `paratehis`=**smallint**, `parinausudias`/`parsenusudias`/`parvalidadereceita{simples,usocon,conesp}`=**smallint** → `Integer`+`@JdbcTypeCode(SMALLINT)`;
`parsecrcep`=**char(8)**, `parsecrfone1/2`=**char(20)** → CHAR trim; `parsecr/end`=varchar(50), `parsecremail`=varchar(70); `parempcod`=integer PK. Mapeamento bate 1:1 com a DB viva → `validate` passa no cutover.
