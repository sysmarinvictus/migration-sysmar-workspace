# SLICE-SPEC — SAU_LOG (Log do Sistema / Auditoria)

> **Wave-0, cross-cutting.** NÃO é uma tela CRUD: é a **trilha de auditoria** do sistema. O entregável
> é tornar o módulo `common/audit` (`AuditService`, hoje **só log**) **persistente** na tabela física
> `SAU_LOG`, assumindo o papel do procedimento legado `psau_inc_log` — chamado após cada INS/UPD/DLT em
> ~32 arquivos do legado. Tabela viva tem **~7,7 milhões de linhas** (append-only). Contém colunas PHI
> (paciente) que o caminho de escrita legado **sempre deixa vazias** — e que o app moderno também NÃO
> deve preencher (LGPD). Extraído 2026-06-22 (gx-extractor + gx-rule-miner + gx-schema-mapper; schema
> verificado contra o banco vivo).

```yaml
slice: SAU_LOG
domain: auditoria                 # → módulo common/audit (NÃO um feature /api CRUD)
description: Log do Sistema
wave: 0
complexity: M                     # sau_log_impl.java 187 KB; 13 regras
primary_table: SAU_LOG
constellation:
  transaction: sau_log.java       # stub
  impl:        sau_log_impl.java  # 187 KB — form CRUD de registro único (UPDATE/DELETE de log existe! ver R11)
  bc:          sau_log_bc.java    # 103 KB — Business Component (valida FK empresa/usuário; bypassed pelo write real)
  sdt:         SdtSAU_LOG.java    # + StructSdtSAU_LOG.java
  insert_proc: psau_inc_log.java  # 7 KB — **o caminho de escrita do sistema** (contrato em R1)
  label_proc:  psau_bus_nometransacao.java  # LogTab → rótulo PT (apenas display, R12)
  workwith: null                  # não há work-with/viewer legado
  view: null
  prompts: []

# Colunas físicas = schema vivo (autoritativo). PK COMPOSTA de 4 colunas. logkey é CHAR(50).
fields:
  - { gx: LogEmpCod,           name: empresaCodigo,      type: int,       pk: true,  nullable: false, note: "tenant/empresa (do contexto de segurança)" }
  - { gx: LogDat,              name: dataHora,           type: timestamp, pk: true,  nullable: false, note: "server now() no momento da escrita; parte da PK (risco de colisão — R6)" }
  - { gx: LogUsuCod,           name: usuarioCodigo,      type: int,       pk: true,  nullable: false, note: "usuário que agiu (do contexto)" }
  - { gx: LogKey,              name: chaveRegistro,      type: char, len: 50, pk: true, nullable: false, note: "CHAR(50)→@JdbcTypeCode(CHAR). PK do registro auditado, stringificada" }
  - { gx: LogOpe,              name: operacao,           type: string, len: 3,  nullable: true,  note: "INS/UPD/DLT/DSP (Gx_mode)" }
  - { gx: LogTab,              name: tabela,             type: string, len: 31, nullable: false, note: "recebe o NOME DO PROGRAMA/transação (ex. SAU_ESP), não o nome SQL — R5" }
  - { gx: UsuCod,              name: usuarioCodigoRef,   type: int,       nullable: true,  note: "= LogUsuCod (duplicado pelo proc); índice isau_log5" }
  - { gx: LogSituacao,         name: situacao,           type: string, len: 50, nullable: true,  note: "legado grava '' " }
  - { gx: LogHistorico,        name: historico,          type: text,      nullable: true,  phi: true, note: "texto livre; legado grava ''; pode conter PHI → manter NULL" }
  - { gx: logNomeProfissional, name: nomeProfissional,   type: string, len: 50, nullable: true,  note: "legado grava ''; NÃO preencher" }
  - { gx: logProPesCod,        name: profissionalCodigo, type: long,      nullable: true,  note: "legado hardcode 0 (defeito R7) → moderno grava o id real quando conhecido" }
  - { gx: lognomePaciente,     name: nomePaciente,       type: string, len: 50, nullable: true,  phi: true, note: "PHI; legado grava ''; NUNCA preencher (LGPD)" }
  - { gx: logPacPesCod,        name: pacienteCodigo,     type: long,      nullable: true,  phi: true, note: "PHI; legado hardcode 0; manter NULL salvo decisão do DPO" }
  - { gx: logUniCod,           name: unidadeCodigo,      type: int,       nullable: true,  note: "legado hardcode 0 (R7) → moderno grava unidade real quando conhecida" }
keys:
  pk: [LogEmpCod, LogDat, LogUsuCod, LogKey]   # composta; sau_log_pkey (ordem confirmada)
  unique: []
  indexes: [{ index: isau_log5, cols: [UsuCod] }]
  fks: []   # ZERO FKs físicas na tabela viva (relações são lógicas/snapshot)
  soft_refs:
    - { col: LogUsuCod,   ref: SAU_USU, migrated: false, wave: 0 }
    - { col: logProPesCod, ref: SAU_PRO, migrated: true,  note: "snapshot; proc zera" }
    - { col: logUniCod,    ref: SAU_UNI, migrated: true,  note: "snapshot; proc zera" }
    - { col: logPacPesCod, ref: SAU_PAC, migrated: false, wave: 6, note: "PHI; snapshot; proc zera" }
    - { col: LogEmpCod,    ref: SYS_EMP, migrated: false, note: "tenant" }

# O caminho de ESCRITA não precisa dessas tabelas migradas: empresa e usuário vêm do contexto de
# segurança (JWT). Os demais ids são snapshot que o legado zera.
depends_on: [SYS_EMP, SAU_USU]   # ambos não migrados; valores vêm do contexto, não de lookup FK.
# SAU_LOG é ele próprio a fundação Wave-0 da qual quase todas as outras fatias dependem (auditoria).

endpoints:
  # ESCRITA = interna, via common/audit/AuditService.record(action, entity, entityId). NÃO é POST público.
  - { method: GET, path: /api/admin/auditoria, status: proposed, role: ADMIN_AUDITORIA,
      note: "OPCIONAL/NOVO (não existe tela legada). Consulta read-only sobre SAU_LOG (filtro por
             tabela/usuário/data/chave, paginado). Admin-gated + auto-auditado. Fora de paridade estrita." }
  # SEM POST/PUT/DELETE públicos. Trilha é append-only (R11) — sem update/delete via API.

phi_fields: [nomePaciente, pacienteCodigo, historico]

auth:
  write: { internal_only: true, note: "AuditService grava sob o contexto do usuário; não é invocável pelo usuário (R9)" }
  read:  { roles_required: [ADMIN_AUDITORIA], note: "sem permissão SAU_LOG encontrada em AcessaModulo.xml; o
           viewer legado (sau_log_impl) é gated por pisauthorized('SAU_LOG', mode) (R10). Confirmar papel (OQ4).
           Qualquer leitura sobre 7,7M linhas com PHI deve ser auth-gated e auto-auditada." }

parity:
  legacy_base: "(sem tela legada — paridade é do CONTRATO DE ESCRITA, não de HTTP)"
  scenarios:
    - "após INSERT em uma fatia migrada (ex. SAU_ESP), existe 1 linha SAU_LOG com LogOpe=INS, LogTab=<programa>, LogKey=<pk str>, LogUsuCod=<ator>, LogEmpCod=<tenant>, LogDat≈now"
    - "após UPDATE → LogOpe=UPD; após DELETE → LogOpe=DLT"
    - "a linha de auditoria NÃO contém nome de paciente/profissional (colunas PHI nulas) — melhoria LGPD vs legado (que já gravava '')"
    - "logProPesCod/logUniCod recebem ids REAIS quando conhecidos (corrige o hardcode 0 do legado, R7) — divergência intencional"
    - "trilha é append-only: nenhuma rota atualiza/exclui linhas de SAU_LOG (melhoria vs legado, R11)"
    - "dois eventos no mesmo tick para mesmo tenant/usuário/chave: ambos persistem (sem colisão de PK — R6)"

status: tested   # backend + testes gerados & verdes 2026-06-28. Suite completa: 263 unit + 279 IT, 0F/0E (84 skipped=stubs paridade).
# +30 testes de auditoria. migration-reviewer: SEM BLOCK; apto a 'tested'. Frontend (viewer admin) DEFERIDO (novo, não-paridade).
# Entregável: AuditService estendido para PERSISTIR em SAU_LOG (LogAuditoria @IdClass, CHAR(50) logkey, repo append-only),
# fail-safe (REQUIRES_NEW + catch-all → falha de auditoria NUNCA quebra a transação de negócio), toggle audit.persist.enabled,
# LGPD (sem nomes/PHI; ids NULL e não 0), retry de colisão +1µs (R6), endpoint /api/admin/auditoria admin-gated + auto-auditado.
# Fecha a lacuna de auditoria adiada em TODAS as fatias migradas (SAU_ESP/SAU_IMP/SAU_PRO…) — sem mudar assinaturas.
#
# CHECKLIST DE CUTOVER (gates p/ 'verified' — nenhum é defeito de código):
#  1. PARITY: /verify-parity SAU_LOG (7 cenários stub @Disabled; contrato de ESCRITA, não HTTP). dir parity/SAU_LOG/ ainda não existe.
#  2. DPO (OQ1): assinar a minimização — colunas PHI mantidas NULL (já implementado e provado por teste negativo).
#  3. REGULATÓRIO (OQ2): retenção/imutabilidade + NOVO: nuance de ordem de commit — o REQUIRES_NEW grava a linha ANTES do commit
#     do negócio; um rollback do negócio deixa linha de auditoria órfã (over-log; legado era After-Trn pós-commit). Divergência
#     intencional (over-log é o lado seguro) — decidir manter ou mover record() para hook after-commit.
#  4. R7 PARCIAL: prof/unidade gravados como NULL (honesto, melhor que o 0 do legado) mas a meta "trilha consultável por
#     profissional/unidade" NÃO foi entregue (a assinatura record(action,entity,id) não carrega esse contexto) → enriquecer depois.
#  5. R12 não implementado (rótulos PT logDescricaoOperacao/logContexto no DTO de leitura) → junto do viewer admin deferido.
#  6. OQ4: leitura gated por hasRole('ADMIN') (coarse) em vez de ADMIN_AUDITORIA — confirmar granularidade p/ viewer de 7,7M linhas.
#  7. PERF: +1 conexão (REQUIRES_NEW) por mutação em todas as fatias — documentar folga do pool antes de rollout amplo.
status_was: specced

open_questions:
  - id: OQ1
    topic: "LGPD — minimização (assinatura do DPO)"
    desc: "lognomepaciente/logpacpescod (PHI) e loghistorico podem conter dados sensíveis. O legado já os
           deixa vazios no caminho de escrita. Confirmar com o DPO que o AuditService moderno mantém essas
           colunas NULL (registra só op/tab/key/usuário/timestamp/empresa + ids reais). Verificar se há OUTRO
           writer que popula nomes (ver OQ5)."
  - id: OQ2
    topic: "Imutabilidade / retenção (assinatura regulatória)"
    desc: "O legado PERMITE UPDATE/DELETE de linhas de auditoria (R11) e a PK baseada em tempo arrisca colisão
           (R6). O store moderno deve ser append-only (sem update/delete) e tratar colisão por timestamp de alta
           precisão + retry — SEM coluna surrogate (regra Fase-1; surrogate = mudança de schema Fase-2 com sign-off).
           Confirmar política de retenção/expurgo governado."
  - id: OQ3
    topic: "Origem do tenant (LogEmpCod)"
    desc: "Como o app moderno deriva empresa/tenant para a linha de auditoria — single-tenant (MunCod=411420 no
           ambiente de paridade) ou do principal? Definir antes de ligar o AuditService."
  - id: OQ4
    topic: "Papel de acesso à leitura"
    desc: "Nenhuma permissão SAU_LOG em AcessaModulo.xml; o viewer legado usa pisauthorized('SAU_LOG'). Confirmar
           qual papel concreto pode LER a trilha (proposto ADMIN_AUDITORIA) e que usuários clínicos NÃO podem."
  - id: OQ5
    topic: "Outros writers de SAU_LOG?"
    desc: "psau_inc_log é o caminho INS do sistema, mas confirmar (grep no tree por 'INSERT INTO SAU_LOG' /
           sau_log_bc.insert) que não há outro writer populando loghistorico/nomes — muda a superfície LGPD."
  - id: OQ6
    topic: "Formato de LogKey por transação"
    desc: "LogKey é a PK do registro afetado formatada como string, específica por transação (PKs compostas
           concatenam). Ao ligar o AuditService em cada fatia, capturar o formato exato da chave para manter as
           linhas joinable com as do legado durante a transição strangler."
  - id: OQ7
    topic: "UsuCod duplicado"
    desc: "Legado grava UsuCod = LogUsuCod. Preservar a redundância ou deixar NULL no writer moderno?"
```

## Regras mineradas (de psau_inc_log / sau_log_bc / sau_log_impl — citação à linha; confiança anotada)

- **R1** (side-effect, high) — Cada commit emite 1 linha em SAU_LOG via `psau_inc_log` (After-Trn). INSERT grava 14 colunas; **só 6 carregam dados reais**, 8 são literais hardcoded: `VALUES(?,?,?,?,?,?,?, 0,'','', 0,'', 0,'')`. Reais: LogEmpCod, LogDat, LogUsuCod, LogKey, LogOpe, LogTab, UsuCod(=LogUsuCod). src `psau_inc_log.java:185, 92-102`. → `AuditServiceTest#emitsOneAuditRowPerCommittedTransaction`.
- **R2** (derivation, high) — LogEmpCod e LogUsuCod vêm do **Context** (tenant + usuário). UsuCod=LogUsuCod. Moderno: do contexto de segurança, nunca do input. src `sau_esp_impl.java:2160-2162, psau_inc_log.java:94-96`.
- **R3** (derivation, high) — LogDat = `serverNow()` no After-Trn (datetime, parte da PK). src `sau_esp_impl.java:2150, psau_inc_log.java:93,206`.
- **R4** (derivation, high) — LogOpe = Gx_mode: INS/UPD/DLT/DSP/"". Moderno: enum {INSERT,UPDATE,DELETE,VIEW}. src `sau_esp_impl.java:2163, sau_log_bc.java:251, psau_inc_log.java:97`.
- **R5** (derivation, high) — **LogTab recebe o NOME DO PROGRAMA/transação (ex. "SAU_ESP"), não o nome SQL**; LogKey = PK afetada stringificada (`GXutil.trim(GXutil.str(A27EspCod,10,0))`). src `sau_esp_impl.java:2155,2164, 1309/3128`.
- **R6** (validation, high) — **PK composta inclui LogDat (tempo)** → dois eventos mesmo tenant/usuário/chave no mesmo tick **colidem** (status 1 / GXM_noupdate). Risco de perda de auditoria. Moderno: timestamp de alta precisão + retry; SEM surrogate (regra Fase-1). src `psau_inc_log.java:185,104-108`. → `#concurrentSameInstantEventsBothPersisted`.
- **R7** (audit, high) — **DEFEITO A CORRIGIR:** write path zera logProPesCod=0, logPacPesCod=0, logUniCod=0 → a trilha nunca registra o profissional/unidade reais. Moderno DEVE gravar o id real do ator/unidade (não zeros), para a trilha ser consultável por ator. src `psau_inc_log.java:185`. → `#recordsRealActorAndUnitIdsNotZeroPlaceholders`.
- **R8** (audit, high, LGPD) — write path grava lognomePaciente='', lognomeProfissional='', LogSituacao='', LogHistorico=''. **Moderno NÃO deve armazenar nomes de paciente/profissional** (minimização LGPD); referenciar sujeitos só por id (R7). src `psau_inc_log.java:185`. → `#auditRowContainsNoPatientOrProfessionalNames`.
- **R9** (authorization, high) — `psau_inc_log` é **interno, sem auth** (GXProcedure, faz commit próprio). Moderno: AuditService é componente interno, sem endpoint exposto. src `psau_inc_log.java:14-178,126`.
- **R10** (authorization, high) — o viewer `sau_log_impl` é gated: sem login → `hnotauthorizeduser`; `pisauthorized("SAU_LOG", mode)`==0 → `hnotauthorizedprompt`. Ler a trilha exige autoridade explícita (AUDIT_VIEW/ADMIN_AUDITORIA). src `sau_log_impl.java:1004-1027`.
- **R11** (side-effect, high) — `sau_log_impl` é **CRUD completo**, com cursores UPDATE (T001215) e DELETE (T001216) sobre SAU_LOG. **O legado permite editar/excluir linhas de auditoria** — fraqueza de integridade. Moderno DEVE ser **append-only** (melhoria intencional). src `sau_log_impl.java:3269,3270`. → `#auditTrailIsAppendOnly`.
- **R12** (formula, high) — derivações de display (não persistidas): logDescricaoOperacao (INS→Inserção…), logContexto via `psau_bus_nometransacao` (mapeia nome de programa→rótulo PT), logDescricaoContexto. Replicar no DTO de leitura, não na entidade. src `sau_log_bc.java:251-256, psau_bus_nometransacao.java:56-107`.
- **R13** (validation, medium) — o **BC** (`sau_log_bc`) valida FK: LogEmpCod→SYS_EMP ("Não existe 'Empresa'") e UsuCod≠0→SAU_USU. **Mas `psau_inc_log` faz SQL cru e ignora o BC** → checagens mortas no caminho real. Informacional: no moderno, tenant/usuário são válidos por construção (vêm do contexto). src `sau_log_bc.java:246-247,271-277`.

## Mapeamento JPA (verificado contra o banco vivo)

Pacote alvo: `br.gov.mandaguari.saude.common.audit`. Entidade **`LogAuditoria`** `@Entity @Table(name="SAU_LOG")` `@IdClass(LogAuditoriaId.class)`.

**`LogAuditoriaId`** (POJO Serializable, equals/hashCode nos 4 campos): `empresaCodigo:Integer, dataHora:LocalDateTime, usuarioCodigo:Integer, chaveRegistro:String`.

| GX col | campo | tipo | anotações |
|--------|-------|------|-----------|
| logempcod | empresaCodigo | Integer | `@Id @Column(name="logempcod", nullable=false)` |
| logdat | dataHora | LocalDateTime | `@Id @Column(name="logdat", nullable=false)` |
| logusucod | usuarioCodigo | Integer | `@Id @Column(name="logusucod", nullable=false)` |
| logkey | chaveRegistro | String | `@Id @JdbcTypeCode(Types.CHAR) @Column(name="logkey", nullable=false, length=50)` |
| logope | operacao | String | `@Column(name="logope", length=3)` |
| logtab | tabela | String | `@Column(name="logtab", nullable=false, length=31)` |
| usucod | usuarioCodigoRef | Integer | `@Column(name="usucod")` (índice isau_log5) |
| logsituacao | situacao | String | `@Column(name="logsituacao", length=50)` |
| loghistorico | historico | String | `@Column(name="loghistorico")` — PHI, NULL na escrita |
| lognomeprofissional | nomeProfissional | String | `@Column(name="lognomeprofissional", length=50)` |
| logpropescod | profissionalCodigo | Long | `@Column(name="logpropescod")` |
| lognomepaciente | nomePaciente | String | `@Column(name="lognomepaciente", length=50)` — PHI, NULL |
| logpacpescod | pacienteCodigo | Long | `@Column(name="logpacpescod")` — PHI, NULL |
| logunicod | unidadeCodigo | Integer | `@Column(name="logunicod")` |

**Sem `@ManyToOne`** — zero FKs físicas; ids ficam escalares (evita acoplar tabelas PHI/não migradas ao agregado de auditoria).
**`logkey` CHAR(50) é membro da PK** → `@JdbcTypeCode(Types.CHAR)` é obrigatório (senão o bpchar com padding quebra o match da PK).

**Repositório append-only:** NÃO estender `JpaRepository`/`CrudRepository` (expõem delete/update). Definir interface estreita só com `save` (insert) + consultas de leitura. Preferir `Persistable<LogAuditoriaId>` com `isNew()=true` (ou INSERT nativo) para garantir INSERT puro e evitar o SELECT pré-insert. Nunca `@Modifying` UPDATE/DELETE nesta tabela.

## Plano Flyway

- **Produção:** SAU_LOG já existe (~7,7M linhas) → `ddl-auto=validate`, **sem ALTER/DDL**.
- **Testcontainers/baseline:** SAU_LOG está **ausente** do `V1__baseline.sql` → adicionar o CREATE TABLE abaixo (ou um `V<n>__sau_log_baseline.sql` se V1 já tiver checksum travado).

```sql
CREATE TABLE IF NOT EXISTS SAU_LOG (
    logempcod           integer                     NOT NULL,
    logdat              timestamp without time zone NOT NULL,
    logusucod           integer                     NOT NULL,
    logope              varchar(3),
    logtab              varchar(31)                 NOT NULL,
    logkey              char(50)                    NOT NULL,
    usucod              integer,
    logsituacao         varchar(50),
    loghistorico        text,
    lognomeprofissional varchar(50),
    logpropescod        bigint,
    lognomepaciente     varchar(50),
    logpacpescod        bigint,
    logunicod           integer,
    CONSTRAINT sau_log_pkey PRIMARY KEY (logempcod, logdat, logusucod, logkey)
);
CREATE INDEX IF NOT EXISTS isau_log5 ON SAU_LOG (usucod);
```

## Nota de integração (o entregável real)
`migrate-slice SAU_LOG` deve **estender `AuditService.record(action, entity, entityId)`** para PERSISTIR uma linha SAU_LOG (mapeando action→logope, entity→logtab, entityId→logkey, ator→logusucod, now→logdat, tenant→logempcod), além do log atual — append-only, sem nomes PHI, com ids reais (corrigindo R7). Como `AuditService` já é chamado por todas as fatias migradas (SAU_ESP, SAU_IMP, SAU_PRO…), ligar a persistência aqui **fecha a lacuna de auditoria adiada em todas elas** — sem mudar as assinaturas dos chamadores.
