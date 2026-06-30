# SLICE-SPEC — SAU_RBAC cluster (Programas + Permissões) · **Wave-0 fine-grained authorization**

> Migrated as one coordinated group (BACKLOG flags the RBAC tables as a cyclic cluster). Covers the
> **program catalog** (SAU_PRGGRP, SAU_PRG) and the **permission matrices** (SAU_PRFCON per-profile,
> SAU_USUCON per-user), and realizes the GeneXus `pisauthorized` precedence engine (SAU_USU **R8/R2/R5**)
> as `PermissionResolver`. This is the deferred "fine RBAC" that SAU_USU's coarse roles pointed to.
> Built 2026-06-30 (schemas live-DB-verified; not via the 3-agent extract — coded directly from the
> SAU_USU rule citations + live introspection, given the cluster's simplicity).

```yaml
slice: SAU_RBAC
domain: programa + autorizacao      # packages …/programa/ (catalog) + …/autorizacao/ (perms + resolver)
description: "Programas do Sistema + Controle de Acesso (perfil/usuário)"
wave: 0
complexity: L                        # 4 tables; the value is the resolver, not the CRUD
tables:
  - { table: SAU_PRGGRP, rows: 28,      cols: 2, pk: [GrpCod],          role: "grupo de programas (leaf)" }
  - { table: SAU_PRG,    rows: 1240,    cols: 6, pk: [PrgCod],          role: "programa (PrgCod = VARCHAR key; permission space)" }
  - { table: SAU_PRFCON, rows: 12287,   cols: 6, pk: [PrfCod,PrfPrgCod], role: "permissão por PERFIL (tier primário)" }
  - { table: SAU_USUCON, rows: 1514426, cols: 6, pk: [UsuCod,PrgCod],    role: "permissão por USUÁRIO (tier fallback)" }
depends_on:
  - { table: SAU_USU, migrated: true, role: "resolver lê UsuSysmar/UsuPrfCod/UsuBloq" }
  - { table: SAU_PRF, migrated: true, role: "resolver valida o perfil (existsById)" }

# ── ENTITIES (live-DB authoritative) ──
entities:
  GrupoPrograma:   "@Table(SAU_PRGGRP); id=GrpCod (Integer), nome=GrpNom (varchar50)"
  Programa:        "@Table(SAU_PRG); id=PrgCod (VARCHAR30, client-supplied), nome=PrgNom(100), grupoId=GrpCod,
                    admin/medico=PrgAdm/PrgMed (Short @SMALLINT), acessoPublico=PrgAcessoPub (Boolean)"
  PerfilPermissao: "@Table(SAU_PRFCON) @IdClass(PerfilPermissaoId{perfilId,programaId}); flags
                    PrfPrgCon/Inc/Alt/Exc (Short @SMALLINT); granted(Mode)"
  UsuarioPermissao:"@Table(SAU_USUCON) @IdClass(UsuarioPermissaoId{usuarioId,programaId}); flags
                    UsuCon/Inc/Alt/Exc (Short @SMALLINT); granted(Mode)"

endpoints:
  # catalog (SAUDE_ADMIN; lookup authenticated)
  - "GET/POST/PUT/DELETE /api/programas (+ /grupos, /lookup)"
  # permission grids + resolver (SAUDE_ADMIN)
  - "GET/PUT /api/autorizacao/perfis/{perfilId}/permissoes[/{programaId}]"
  - "GET/PUT /api/autorizacao/usuarios/{usuCod}/permissoes[/{programaId}]"
  - "GET     /api/autorizacao/check?usuCod=&programaId=&mode=  (resolve effective permission)"

phi_fields: []                       # RBAC catalogs/matrices — no PHI
auth: { roles_required_admin: [SAUDE_ADMIN] }

status: tested   # backend + testes (16 unit + 11 IT) verdes 2026-06-30; revisão SEM BLOCK. Suite 296/0F/0E.
# Revisão (migration-reviewer): motor de autorização CORRETO e fail-closed — nenhum caminho libera
# usuário negado/bloqueado/desconhecido/principal-não-numérico; precedência de perfil garantida (nunca OR);
# oráculo /check é SAUDE_ADMIN-only. 2 FLAGs corrigidos: (a) /check agora audita (AUTHZ_CHECK); (b) IT do
# upsert per-user adicionado. FLAGs remanescentes → OQ4 (paridade vs pisauthorged incl. blocked-vs-sysmar) e
# OQ5 (PrgAcessoPub não consultado — fail-closed seguro, confirmar semântica) ANTES de fazer o wiring (OQ1).
status_was: pending
```

## Regras realizadas

- **R8/R2/R5 — `PermissionResolver.can(usuCod, programaId, mode)`** (o motor `pisauthorized`):
  1. usuário desconhecido → nega; 2. **bloqueado (UsuBloq=1) → nega** (R5); 3. **UsuSysmar=true → libera tudo**
  (R2, só o flag); 4. **perfil válido** (UsuPrfCod>0 ∈ SAU_PRF) → **SAU_PRFCON** (perfil tem **precedência**,
  não é OR); 5. senão → **SAU_USUCON** (fallback por-usuário); 6. sem linha → nega. Modos CON/INC/ALT/EXC.
  Bean `@Component("rbac")` → `@PreAuthorize("@rbac.can(authentication,'PRG','ALT')")` (principal=UsuCod).
  Cobertura: `PermissionResolverTest` (11) + `RbacControllerIT` (10, resolução end-to-end + upsert composite-PK).
- **Catálogo SAU_PRG/SAU_PRGGRP** — CRUD; PrgCod único (Conflict); delete bloqueado se referenciado por
  SAU_PRFCON/SAU_USUCON; audit em todo CRUD. `ProgramaServiceTest` (5).
- **Grids SAU_PRFCON/SAU_USUCON** — leitura + upsert (PUT) com PK composta; audit `PERM_SET`. `AutorizacaoService`.

## Plano Flyway (V1 baseline — confirmado vs DB vivo)
SAU_PRGGRP, SAU_PRG (PrgCod varchar30, GrpCod, PrgAdm/PrgMed smallint, PrgAcessoPub boolean), SAU_USUCON
(PK UsuCod,PrgCod + 4 flags) adicionados; SAU_PRFCON.PrfPrgCod corrigido para VARCHAR(30) (era o stub int do
slice SAU_PRF). Produção já tem as tabelas → `validate`.

## 🔎 OPEN QUESTIONS
- **OQ1 (wiring por-endpoint):** o motor está pronto e exposto via `@rbac.can(...)`, mas os endpoints já
  migrados continuam com **RBAC coarse** (SAU_USU OQ7). Mapear cada endpoint moderno → PrgCod GeneXus +
  modo é trabalho **endpoint-a-endpoint** (diferido; precisa do mapa tela-legada→rota-nova).
- **OQ2 (módulo R9 / acessomodulo):** o gate de MÓDULO por (UsuCod,UniCod) via SAU_USUUNI (SAU_USU R9) NÃO
  está aqui — fica para a fatia SAU_USUUNI (Wave-4, login-por-unidade).
- **OQ3 (volume SAU_USUCON):** 1.5M linhas — a UI de manutenção por-usuário deve ser sob demanda (por
  usuário), nunca listagem global. Resolver faz lookup por PK composta (O(1)), sem varredura.
- **OQ4 (paridade):** sem `parity/` ainda — `/verify-parity` deve confirmar a resolução vs `pisauthorized`
  legado para uma amostra de (usuário, programa, modo), incluindo **blocked-vs-sysmar** (R5 antes de R2) e
  os 3 tiers. **Obrigatório ANTES do wiring (OQ1).**
- **OQ5 (PrgAcessoPub):** o flag "acesso público" do programa é persistido mas o resolver **não o consulta**
  (fail-closed, mais estrito que o legado — seguro). Se o `pisauthorized` legado libera quando
  `PrgAcessoPub=1` independente das matrizes, adicionar um tier-0 e testar paridade. Confirmar semântica.
