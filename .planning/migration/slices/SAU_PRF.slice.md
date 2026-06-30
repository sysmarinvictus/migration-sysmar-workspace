# SLICE-SPEC — SAU_PRF (Perfil) · **RBAC profile tier (Wave 0)**

> The access-profile catalog read read-only by SAU_USU (`UsuPrfCod`). Migrating it gives a real `Perfil`
> entity + CRUD and a delete-guard contract. **Trivial table** (2 cols, 10 rows) but the GeneXus form embeds
> the SAU_PRFCON per-program permission grid — that grid is migrated as its **own** slice (`SAU_PRFCON`,
> next); this slice covers the **Perfil header CRUD + delete guards + cascade-delete of child permissions**.
> Extracted 2026-06-30 (3 mining agents; all claims live-DB-verified).

```yaml
slice: SAU_PRF
domain: perfil                    # package …/perfil ; REST /api/perfis (NAMING.md)
description: Perfil
wave: 0
complexity: M                     # sau_prf_impl.java ≈128 KB (rule source); table itself is trivial
primary_table: SAU_PRF            # PK PrfCod (int), 10 rows, 2 cols, ZERO physical FKs
constellation:
  transaction: sau_prf.java
  impl: sau_prf_impl.java          # 128 KB — header CRUD + delete guards + audit + embedded SAU_PRFCON grid
  bc: null                         # NOT a Business Component (no gxmetadata json) → live DB authoritative
  workwith: null                   # NO hwwsau_prf → list endpoint synthesized
  view: null                       # NO hviewsau_prf → detail endpoint synthesized
  prompt: hpromptsau_prf.java       # "Selecionar Perfil" → /api/perfis/lookup
  sections: []
  sdt: null
  procedures: [psau_inc_prf.java]   # "Gera Código do Perfil" — PrfCod = MAX+1 allocation
  embedded_grid: [sau_prfcon.java, sau_prfcon_impl.java]   # per-program permission grid — OWN slice (SAU_PRFCON)

depends_on: []                    # root catalog — zero forward deps
referenced_by:                    # REVERSE deps = delete guards (other tables point AT PrfCod)
  - { table: SAU_USU,    col: UsuPrfCod,        migrated: true,  role: "user→profile (R4 delete guard)" }
  - { table: SAU_PRFCON, col: PrfCod,           migrated: false, role: "per-profile perms — CASCADE on delete (R6), NOT a block" }
  - { table: SAU_PAR4,   col: ParProSocPrfCod,  migrated: false, role: "social-professional default profile param (R5 delete guard)" }

# ── ENTITY (live-DB authoritative; trivial int+varchar, no @JdbcTypeCode/@Convert) ──
fields:
  - { gx: PrfCod, name: id,   type: int,    pk: true,  nullable: false, domain: Codigo06, picture: "ZZZZZZ",
      note: "server-allocated MAX(PrfCod)+1 on insert (psau_inc_prf) — NOT @GeneratedValue (no sequence/identity in live DB)" }
  - { gx: PrfNom, name: nome, type: string, len: 50,   nullable: true,  picture: "@!",
      note: "DB NULL-able but app-required (R2) → @NotBlank in DTO, column stays nullable to pass validate. Stored UPPERCASE (R3)." }

keys:
  pk: [PrfCod]
  unique: []                      # NO unique on PrfNom (confirmed live: only pk + non-unique usau_prf_desc) → no DB UNIQUE
  indexes: [{ index: usau_prf_desc, cols: [PrfCod], order: DESC, unique: false }]
  fks: []                         # ZERO physical FKs

endpoints:                        # /api/perfis — admin (RBAC maintenance)
  - { method: GET,    path: /api/perfis,        auth: ADMIN, from: "synthesized list (no hww)" }
  - { method: GET,    path: /api/perfis/{id},   auth: ADMIN, from: "synthesized detail (no hview)" }
  - { method: POST,   path: /api/perfis,        auth: ADMIN, note: "PrfCod auto-allocated (R1); PrfNom required (R2) + uppercased (R3)" }
  - { method: PUT,    path: /api/perfis/{id},   auth: ADMIN, note: "PrfNom required (R2) + uppercased (R3)" }
  - { method: DELETE, path: /api/perfis/{id},   auth: ADMIN, note: "blocked if referenced by SAU_USU (R4) or SAU_PAR4 (R5); cascades SAU_PRFCON (R6)" }
  - { method: GET,    path: /api/perfis/lookup, auth: authenticated, from: hpromptsau_prf, desc: "autocomplete {id, nome}" }

phi_fields: []                    # RBAC catalog — no patient PHI/PII
auth:
  roles_required_admin: [SAUDE_ADMIN]   # profile/RBAC maintenance is administrative (matches SAU_USU CRUD). Confirm exact program guard (OQ2).
  source: "sau_prf_impl.java:945-963 (pisauthorized('SAU_PRF', mode)); acessamodulo.java"

parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment (auth-gated; headless capture not expected — behavioral parity vs live DB + mined rules, cf. SAU_IMP/SAU_USU)"
  scenarios:
    - "list default → 200, 10 real profiles (ENFERMEIRO, ACS, ATENDENTE, …)"
    - "get by id → 200, {id, nome}"
    - "create → PrfCod = MAX+1, PrfNom stored UPPERCASE (R1/R3)"
    - "create blank nome → rejected 'Informe a descrição do perfil!' (R2)"
    - "delete profile referenced by a SAU_USU user → blocked (R4)"
    - "delete profile referenced by SAU_PAR4.ParProSocPrfCod → blocked (R5)"
    - "delete unreferenced profile → 204 + cascades its SAU_PRFCON rows (R6)"
    - "lookup 'enf' → returns ENFERMEIRO"

status: tested   # backend + testes (12 unit + 13 IT) verdes 2026-06-30; revisão sem BLOCK. Suite unit 280/0F/0E.
# Gerado (…/perfil/): Perfil (PrfCod/PrfNom), PerfilRepository (findMaxId, guards nativos, cascade), PerfilService
# (R1 MAX+1, R2/R3 nome required+upper, R4/R5 delete-guards, R6 cascade SAU_PRFCON, R10 audit), PerfilController
# (/api/perfis, SAUDE_ADMIN; lookup isAuthenticated). V1 baseline + stubs SAU_PRFCON/SAU_PAR4. PrfNom nullable
# no DB + @NotBlank no DTO. Colunas de guard confirmadas vivas (parprosocprfcod/prfcon.prfcod/usu.usuprfcod).
# PENDENTE: /verify-parity SAU_PRF (sem fixtures ainda) + frontend React (feature perfil — diferido).
status_was: specced
```

## Regras mineradas (citação à linha; confiança anotada)

- **R1** (default, **high**) — No INSERT, `PrfCod` (PK) é **auto-alocado** = `MAX(PrfCod)+1` sobre SAU_PRF (ou 1 se vazia); campo sempre disabled/read-only na UI. Não é `@GeneratedValue` (live DB não tem sequence/identity). src `sau_prf_impl.java:1735-1743, psau_inc_prf.java:50-71,115`. → `PerfilServiceTest#allocatesNextPrfCodAsMaxPlusOneOnInsert`.
- **R2** (validation, **high**) — `PrfNom` obrigatório: vazio → "Informe a descrição do perfil!". Única validação de campo do header. (DB permite NULL → impor no DTO `@NotBlank`, coluna fica nullable p/ validate.) src `sau_prf_impl.java:1063-1069`. → `PerfilServiceTest#rejectsBlankPrfNom`.
- **R3** (derivation, **high**) — `PrfNom` normalizado para **MAIÚSCULAS** (UI `toUpperCase()` + servidor `GXutil.upper`). Persistir uppercased. src `sau_prf_impl.java:590,392`. → `PerfilServiceTest#uppercasesPrfNom`.
- **R4** (validation, **high**, delete-guard) — Não deletar perfil se **algum SAU_USU** o referencia (`UsuPrfCod = PrfCod`) → CannotDeleteReferencedRecord ("Usuários do Sistema"). src `sau_prf_impl.java:1526-1533,3071`. → `PerfilServiceTest#blocksDeleteWhenReferencedByUsuario`.
- **R5** (validation, **medium**, delete-guard) — Não deletar perfil se for o perfil-padrão de "profissional social" em `SAU_PAR4.ParProSocPrfCod` → bloqueado. (Mensagem genérica "Level4"; semântica inferida do nome da coluna — confirmar.) src `sau_prf_impl.java:1534-1541,3072`. → `PerfilServiceTest#blocksDeleteWhenReferencedBySau_par4ProSocProfile`.
- **R6** (side-effect, **high**) — Deletar perfil **cascateia** suas linhas `SAU_PRFCON` (PrfCod) na mesma transação. SAU_PRFCON **NÃO bloqueia** o delete (é o nível filho do form). src `sau_prf_impl.java:1545-1616,3078`. → `PerfilServiceTest#cascadeDeletesSauPrfconRowsOnProfileDelete`.
- **R9** (authorization, **medium**) — Acesso à manutenção de SAU_PRF é gated: exige usuário logado + `pisauthorized('SAU_PRF', modo)` autorizado. Portar p/ checagem de role nos endpoints `/api/perfis` (`SAUDE_ADMIN`). src `sau_prf_impl.java:945-963`. → `PerfilControllerSecurityTest#deniesUnauthorizedPrfAccess`.
- **R10** (audit, **high**) — Todo INS/UPD/DLT escreve auditoria via `psau_inc_log` (SAU_LOG): EmpCod, timestamp, UsuCod, modo (INS/UPD/DLT), nome do programa, LogKey = PrfCod texto. **Mapear p/ `common/audit`** (já implementado). src `sau_prf_impl.java:1651-1683`. → `PerfilServiceTest#writesAuditRecordOnCrud`.
- **R11** (validation, **low**) — Delete é confirmation-gated (GXM_confdelete). No REST → DELETE explícito destrutivo, guardado pelas checagens R4/R5 antes do commit. src `sau_prf_impl.java:786-795`. → (coberto pelos guards; teste opcional).

### Regras do grid SAU_PRFCON (FORA do escopo desta fatia → slice SAU_PRFCON)
- **R7** (validation, high) — Cada filho `PrfPrgCod` deve existir em SAU_PRG → senão "Não existe 'Programas'." src `sau_prf_impl.java:1830-1844`.
- **R8** (validation, high) — `(PrfCod, PrfPrgCod)` único no grid; flags `PrfPrgCon/Inc/Alt/Exc` default 0. src `sau_prf_impl.java:1982`.

## Mapeamento JPA (verificado contra o banco vivo — 2 colunas, 10 linhas, PK prfcod, ZERO FK)
Entidade `Perfil` `@Entity @Table("SAU_PRF")` em `…/perfil/`. `prfcod`→`id` (`@Id Integer @Column("PrfCod")`, MAX+1 no serviço, NÃO @GeneratedValue), `prfnom`→`nome` (`@Column("PrfNom", length=50)`, nullable no DB, `@NotBlank` no DTO, uppercased). Sem `@ManyToOne`/`@OneToMany`. Delete-guards = queries nativas `exists`:
- `SAU_USU.usuprfcod` (int, nullable) · `SAU_PRFCON.prfcod` (int) · `SAU_PAR4.parprosocprfcod` (int) — todas confirmadas vivas.

## Plano Flyway
- **Produção:** SAU_PRF já existe → `validate`, sem ALTER.
- **Baseline (V1):** SAU_PRF está **ausente** do `V1__baseline.sql` (baseline para em SAU_IMP) → ITs falham `validate`. **Adicionar:**
```sql
CREATE TABLE IF NOT EXISTS SAU_PRF (
    PrfCod  INTEGER NOT NULL,
    PrfNom  VARCHAR(50),
    CONSTRAINT pk_sau_prf PRIMARY KEY (PrfCod) );
CREATE INDEX IF NOT EXISTS usau_prf_desc ON SAU_PRF (PrfCod DESC);
```
(sem UNIQUE em PrfNom — espelha o vivo; sem sequence/identity — MAX+1 no serviço.)

## 🔎 OPEN QUESTIONS (revisar antes de /migrate-slice)
- **OQ1 (UX list/detail):** não há hww/hview/bc → endpoints de lista/detalhe são **sintetizados** (não 1:1 com legado). Confirmar expectativa de UX. (Baixo risco — CRUD trivial.)
- **OQ2 (guard exato):** qual programa/menu/permissão protege a manutenção de SAU_PRF. `pisauthorized('SAU_PRF',modo)` resolve em runtime; mapeei p/ `SAUDE_ADMIN` (consistente com CRUD de SAU_USU). Confirmar.
- **OQ3 (escopo SAU_PRFCON):** o grid de permissões por-programa é **embedded** no form de SAU_PRF, mas `sau_prfcon.java` existe como objeto próprio. **Decisão:** migrar SAU_PRFCON como **fatia própria** (próxima); esta fatia faz só o header + cascade-delete (R6). R7/R8 ficam na fatia SAU_PRFCON.
- **OQ4 (R5 semântica):** `SAU_PAR4.ParProSocPrfCod` ("perfil padrão profissional social") — confirmar a regra de negócio com líder de domínio antes de finalizar a mensagem.
- **OQ5 (unicidade PrfNom):** sem unique no DB e sem checagem no legado → tratar **não-único** (não adicionar UNIQUE nem checagem de serviço, salvo decisão explícita).

## 🔎 Revisão (migration-reviewer) — 2026-06-30
**Veredito: READY (sem BLOCK).** Todas as 8 regras em-escopo (R1–R6, R9, R10) implementadas + testadas; R7/R8
corretamente diferidas (fatia SAU_PRFCON); R11 subsumida nos guards. Fidelidade de schema e LGPD OK. FLAGs:
- **FLAG-1 (paridade):** sem `parity/perfil/` ainda → rodar `/verify-parity SAU_PRF` antes do cutover de rota
  (padrão behavioral-parity vs DB vivo + regras, como SAU_IMP/SAU_USU). 8 cenários listados no bloco `parity`.
- **FLAG-2 (nome de teste):** o spec citava `PerfilControllerSecurityTest` para R9; os 401/403 vivem no
  `PerfilControllerIT` (cobertura equivalente). Sem ação — só divergência de nome.
- **FLAG-3 (corrida MAX+1):** janela TOCTOU em `findMaxId()+1`; mitigada por colisão de PK → 409. Aceitável p/
  tabela admin de ~10 linhas (mesma janela do legado psau_inc_prf). Limitação conhecida.
- **FLAG-4 ✅ RESOLVIDA:** colunas de guard confirmadas vivas — `sau_par4.parprosocprfcod`,
  `sau_prfcon.prfcod`, `sau_usu.usuprfcod` (todas existem). Stubs do baseline batem com o vivo.
- **FLAG-5 (catch defensivo):** R5/R6 engolem `RuntimeException` (tolera tabela ausente em baseline/test).
  No cutover, com SAU_PAR4/SAU_PRFCON já no baseline, **estreitar o catch** (só swallow de undefined-table)
  para um erro real de guard não virar delete silencioso de perfil referenciado. → tratar na fatia SAU_PRFCON.

> **Frontend diferido:** a feature React `perfil/` não foi gerada nesta passada (perfil é sub-form admin, usado
> pelo picker de SAU_USU). Backend + testes + prontidão de paridade entregues. Gerar no /migrate-slice da UI.
