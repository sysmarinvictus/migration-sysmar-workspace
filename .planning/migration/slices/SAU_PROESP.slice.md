# SLICE-SPEC — SAU_PROESP (Especialidade do Profissional) · Wave-4 association

> Association SAU_PRO ↔ SAU_ESP (a professional's specialties). Both parents migrated (SAU_PRO tested,
> SAU_ESP verified). Composite-PK slice — same `@IdClass` pattern as SAU_PRFCON/SAU_USUCON (this session).
> Grounded 2026-06-30 (live schema + targeted impl grep). **STATUS: specced — ready for `/migrate-slice`.**

```yaml
slice: SAU_PROESP
domain: profissionalespecialidade      # REST sub-resource: /api/profissionais/{proPesCod}/especialidades
description: Especialidade do Profissional
wave: 4
complexity: M                          # sau_proesp_impl.java ~316 KB
primary_table: SAU_PROESP              # composite PK (ProPesCod bigint, EspCod int), 1904 rows
impl: sau_proesp_impl.java             # rule source
depends_on:
  - { table: SAU_PRO, migrated: true,  col: ProPesCod }
  - { table: SAU_ESP, migrated: true,  col: EspCod }
referenced_by:
  - { table: SAU_IMP, col: "(ProPesCod, EspCod)", role: "DELETE GUARD (R5)" }

# ── ENTITY (live-verified) — @Entity @Table(SAU_PROESP) @IdClass(ProfissionalEspecialidadeId) ──
fields:
  - { gx: ProPesCod,       name: profissionalId,   type: long,  pk: true, nullable: false, fk: SAU_PRO }
  - { gx: EspCod,          name: especialidadeId,  type: int,   pk: true, nullable: false, fk: SAU_ESP }
  - { gx: ProEspPri,       name: prioritario,      type: short, jdbc: SMALLINT, note: "0/1 checkbox 'prioritário'; expose as boolean; default 0 (R4)" }
  - { gx: ProEspSit,       name: situacao,         type: short, jdbc: SMALLINT, note: "1=Ativo; default 1 on insert (R3)" }
  - { gx: ProEspAgeManQtd, name: agendaManhaQtd,   type: short, jdbc: SMALLINT }
  - { gx: ProEspAgeTarQtd, name: agendaTardeQtd,   type: short, jdbc: SMALLINT }
  - { gx: ProEspAgeNoiQtd, name: agendaNoiteQtd,   type: short, jdbc: SMALLINT }
keys: { pk: [ProPesCod, EspCod], indexes: [isau_proesp2 (EspCod)] }

endpoints:   # sub-resource of profissional; SAUDE_CADASTRO
  - { method: GET,    path: /api/profissionais/{proPesCod}/especialidades }
  - { method: POST,   path: /api/profissionais/{proPesCod}/especialidades,          note: "add specialty (R1/R2 FK, R7 unique, R3/R4 defaults)" }
  - { method: PUT,    path: /api/profissionais/{proPesCod}/especialidades/{espCod},  note: "update flags/agenda" }
  - { method: DELETE, path: /api/profissionais/{proPesCod}/especialidades/{espCod},  note: "R5 blocked if SAU_IMP exists" }
phi_fields: []      # RBAC/clinical association — no patient PHI
auth: { roles_required: [SAUDE_CADASTRO] }
status: tested
status_was: specced
open_questions:
  - id: OQ1-proesp1
    text: >
      The legacy `sau_proesp` transaction manages a SECOND physical table, **SAU_PROESP1** — the
      per-unit / per-period schedule detail: PK (ProPesCod, EspCod, ProEspUniCod, ProEspPer) + weekday
      flags (ProEspDom/Seg/Ter/Qua/Qui/Sex/Sab) + ProEspExpEsus. **R6 ("Informe o Período!") belongs to
      SAU_PROESP1, NOT SAU_PROESP.** This slice migrated ONLY the SAU_PROESP master row (Pri/Sit + the
      Man/Tar/Noi aggregate quotas). SAU_PROESP1 is a DEFERRED sub-slice. Before cutover: confirm whether
      the UI needs the period schedule; if so, backlog `SAU_PROESP1`. Do NOT enforce R6 against SAU_PROESP.
    status: deferred
  - id: OQ2-parity
    text: >
      No parity fixtures yet (`parity/profissionalespecialidade/` absent) — expected at `tested`. Run
      `/verify-parity SAU_PROESP` against the legacy app (MunCod=411420) before cutting the route over;
      do NOT cut over on unit+IT alone.
    status: open
  - id: OQ3-situacao-bounds
    text: >
      `EspecialidadeUpdateRequest.situacao` is a free Short; legacy cmbProEspSit only offers 1=Ativo /
      2=Inativo. Low risk (an out-of-domain value would be accepted). Optionally constrain to {1,2}.
    status: open
```

## Regras mineradas (citação à linha — sau_proesp_impl.java)
- **R1** (validation, high) — ProPesCod obrigatório ("Informe o Código do Profissional!") + deve existir em **SAU_PRO**. src `:1636,4359`. → `ProfEspServiceTest#requiresAndValidatesProfissional`.
- **R2** (validation, high) — EspCod obrigatório ("Informe o Código da Especialidade!") + deve existir em **SAU_ESP**. src `:1680,4434`. → `#requiresAndValidatesEspecialidade`.
- **R3** (default, high) — `ProEspSit` default **1** (Ativo) no insert. src `:259`. → `#defaultsSituacaoToAtivo`.
- **R4** (default, high) — `ProEspPri` é checkbox 0/1 ('prioritário'); default **0**. src `:912,918`. → `#prioritarioDefaultsFalse`.
- **R5** (validation, high, **delete-guard**) — Delete bloqueado se existe **SAU_IMP** para (ProPesCod,EspCod) → "CannotDeleteReferencedRecord 'Impedimento'". Cursor `SELECT ImpCod FROM SAU_IMP WHERE ProPesCod=? AND EspCod=? LIMIT 1`. src `:2447,5462`. → `#blocksDeleteWhenImpedimentoExists`. **⚠ impedimentos NÃO bloqueiam prescrições** (só o vínculo pro-esp).
- **R6** (validation, med) — ⚠ **FORA DE ESCOPO — pertence a SAU_PROESP1** (não a SAU_PROESP). "Informe o Período!" é do subgrid de agenda por unidade/período (SAU_PROESP1: ProPesCod,EspCod,ProEspUniCod,ProEspPer + flags dia-da-semana). src `:2880,5438,5469`. Sub-slice DIFERIDO (ver open_questions OQ1-proesp1). As qtds Man/Tar/Noi desta fatia são os agregados do master, não o schedule.
- **R7** (validation, high) — PK composta única (ProPesCod,EspCod) — não adicionar a mesma especialidade 2× ao profissional (DuplicatePrimaryKey → Conflict).
- **R8** (audit, high) — CRUD → `psau_inc_log`/SAU_LOG → mapear `common/audit`.

## Plano de implementação (próxima sessão — copiar o padrão SAU_PRFCON/USUCON)
1. `…/profissionalespecialidade/domain/`: `ProfissionalEspecialidade` (@IdClass) + `ProfissionalEspecialidadeId{profissionalId(Long), especialidadeId(Integer)}` (equals/hashCode).
2. `repository/ProfissionalEspecialidadeRepository`: `findByProfissionalId(Long)`; native `EXISTS sau_pro`/`EXISTS sau_esp` (R1/R2); native `EXISTS(SELECT 1 FROM sau_imp WHERE propescod=? AND espcod=?)` (R5).
3. `service/`: list; add (R1/R2 FK, R7 existsById→Conflict, R3 sit=1, R4 pri=0, audit CREATE); update (audit UPDATE); remove (R5 guard→Conflict, audit DELETE).
4. `dto/` + `web/ProfissionalEspecialidadeController` (sub-resource, SAUDE_CADASTRO).
5. **Baseline (V1):** SAU_PROESP stub tem só a PK → adicionar (idempotente):
   ```sql
   ALTER TABLE SAU_PROESP ADD COLUMN IF NOT EXISTS ProEspPri SMALLINT;
   ALTER TABLE SAU_PROESP ADD COLUMN IF NOT EXISTS ProEspSit SMALLINT;
   ALTER TABLE SAU_PROESP ADD COLUMN IF NOT EXISTS ProEspAgeManQtd SMALLINT;
   ALTER TABLE SAU_PROESP ADD COLUMN IF NOT EXISTS ProEspAgeTarQtd SMALLINT;
   ALTER TABLE SAU_PROESP ADD COLUMN IF NOT EXISTS ProEspAgeNoiQtd SMALLINT;
   ```
   (SAU_IMP já está no baseline com (ProPesCod,EspCod) → guard IT funciona.)
6. Testes: `ProfEspServiceTest` (R1-R5,R7) + `ProfEspControllerIT` (Testcontainers: add/list/update/delete + delete-guard via SAU_IMP + 401/403). Rodar `mvn -o test` (limpar `target/*.jar.original` antes) + a IT via failsafe.
7. Status → tested; commit app + spec workspace; push (usar `git -C <dir>`). `/verify-parity` depois.

## Notas de contexto
- Nada foi implementado/commitado ainda para esta fatia (só este spec). Baseline stub atual: `SAU_PROESP (ProPesCod, EspCod)` PK apenas.
- Padrão de referência pronto: `…/autorizacao/` (SAU_PRFCON/USUCON @IdClass) e `…/impedimento/` (SAU_IMP, o delete-guard alvo).

## Resultado (2026-06-30 — /migrate-slice)
Backend (entity+IdClass+repo+service+dto+controller) + testes (Service 12, IT 11 Testcontainers) +
frontend (painel de especialidades embutido em ProfissionalDetailPage) implementados. Suíte completa
**405 testes, 0 falhas** (baseline ALTERs aditivos não regrediram nada). OpenAPI regenerado (84 paths,
+4 endpoints SAU_PROESP em `frontend/openapi.json`). `migration-reviewer`: **READY, 0 BLOCK**; 3 FLAGs →
open_questions (OQ1 SAU_PROESP1 diferido, OQ2 parity, OQ3 bounds de situação). Próximo: `/verify-parity`.
