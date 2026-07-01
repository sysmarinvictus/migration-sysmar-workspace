# SLICE-SPEC — SAU_USUUNI (Usuário × Unidade) · Wave-5 · capability matrix

> Per-(user,unit) **capability matrix**: composite PK (UsuCod, UniUsuCod) + **54 native-boolean flags**
> (49 `Bloq*` block + 5 `Per*` permit) + optional `UsuUniEspCod`→SAU_ESP. Both parents migrated (SAU_USU
> Wave-0 tested, SAU_UNI Wave-2 verified). Live-DB introspected 2026-07-01 (54 boolean cols, PK
> usucod,uniusucod). **First Wave-5 slice.**

```yaml
slice: SAU_USUUNI
domain: usuariounidade                 # /api/usuarios/{usuCod}/unidades (sub-resource of usuário)
description: "Usuário × Unidade — per-(user,unit) capability/block matrix"
wave: 5
complexity: L                          # sau_usuuni_impl.java ~510 KB (mostly the 54-flag matrix)
primary_table: SAU_USUUNI              # composite PK (UsuCod, UniUsuCod)
impl: sau_usuuni_impl.java
depends_on:
  - { table: SAU_USU, migrated: true,  col: UsuCod,    note: "user (Wave-0)" }
  - { table: SAU_UNI, migrated: true,  col: UniUsuCod, note: "unit → SAU_UNI.UniCod (Wave-2)" }
  - { table: SAU_ESP, migrated: true,  col: UsuUniEspCod, note: "optional auditor specialty (Wave-3)" }
child_table:
  - { table: SAU_USUUNI1, pk: [UsuCod, UniUsuCod, UsuSetorCod], note: "allowed setores per (user,unit) → SAU_UNISETOR — DEFERRED sub-slice (OQ2)" }
keys: { pk: [UsuCod, UniUsuCod] }
fields:
  - { gx: UsuCod,       name: usuCod, type: Integer, pk: true, fk: SAU_USU }
  - { gx: UniUsuCod,    name: uniCod, type: Integer, pk: true, fk: SAU_UNI.UniCod }
  - { gx: UsuUniEspCod, name: especialidadeCod, type: Integer, nullable: true, fk: SAU_ESP }
  - note: "54 flags — LIVE-VERIFIED native PostgreSQL boolean (NOT smallint). 49 Bloq* (block a module in
           the unit: Tabela/Cadastro/Ambulatorio/Prontuario*/Farmacia/Almox/Agenda*/Lab/.../Rel* report
           families) + 5 Per* (permit: ConsultaDireta/Bnafar/SoaBnafar/AgendaAuditor/AgendaAuditorPcd).
           Full name map in the generator (V11 + entity). Nullable; null = not blocked / not granted."
endpoints:                             # sub-resource of usuário; SAUDE_ADMIN
  - { method: GET,    path: /api/usuarios/{usuCod}/unidades }
  - { method: GET,    path: /api/usuarios/{usuCod}/unidades/{uniCod} }
  - { method: POST,   path: /api/usuarios/{usuCod}/unidades,        note: "grant access (?uniCod=), R2/R4 checks" }
  - { method: PUT,    path: /api/usuarios/{usuCod}/unidades/{uniCod} }
  - { method: DELETE, path: /api/usuarios/{usuCod}/unidades/{uniCod}, note: "R6 unconditional" }
phi_fields: []                         # authorization matrix — no patient data
auth: { roles_required: [SAUDE_ADMIN], note: "legacy gates via SAU_USU program; this table IS an authz source (OQ3)" }
status: tested
status_was: pending
```

## Regras (sau_usuuni_impl.java — validação MÍNIMA, confirmado)
- **R1** UsuCod existe em SAU_USU (`:2721`) → `repo.usuarioExists` (list/create → 404).
- **R2** UniUsuCod existe em SAU_UNI (`:2744`) → `repo.unidadeExists` → 422 "Unidade não existe".
- **R3** UsuUniEspCod opcional; se ≠0 existe em SAU_ESP (`:2758`) → 422 "Não existe Especialidade do usuário".
- **R4** PK composta única (`:2142`) — não conceder 2× a mesma (user,unit) → 409.
- **R5** insert: todos os 54 flags default false/0 (`:5052`) — aqui: flags não enviados ficam **null** (= não bloqueado/não concedido). Nada é bloqueado por padrão.
- **R6** delete incondicional (`:7663`) — sem delete-guard referencial.
- **R7/R8** subgrid SAU_USUUNI1 (setores) — **DIFERIDO** (ver OQ2).
- **R9** campos lookup (UsuNom/UniUsuNom/etc.) read-only/derivados — não persistidos aqui.
- **R10** auditoria (psau_inc_log) → `common/audit` CREATE/UPDATE/DELETE, chave `UsuCod/UniUsuCod`.
- **R11** auth: legado usa a permissão do programa SAU_USU; aqui **SAUDE_ADMIN** (admin de permissões).
- **Confirmado:** SEM required-field "Informe", SEM regras cross-field entre flags — os 54 são booleanos independentes. Não inventar acoplamento "if BloqX then BloqY".

## Persistência (LIVE-introspectado 2026-07-01)
- 54 flags = **boolean nativo** no PG (não smallint — o extractor supôs smallint; a introspecção corrigiu). Mapear como `Boolean` puro (sem converter). UsuUniEspCod = integer nullable.
- **V11__sau_usuuni_matrix.sql**: `ADD COLUMN IF NOT EXISTS <flag> BOOLEAN` ×54 + UsuUniEspCod INTEGER, e **conserta a PK** de (UsuCod) [stub do baseline] → (UsuCod, UniUsuCod) via bloco `DO $$` que só age quando a PK atual é a stub de coluna única (no-op na DB viva, que já é composta).
- Gerados por script (`/tmp/gen_usuuni.py`) p/ consistência dos 54 campos entre entity/dto/service/migration/test.

## Resultado (2026-07-01 — /migrate-slice)
Backend (usuariounidade/: entity @IdClass + 54 flags + dto + repo + service + controller). Testes: Service
10 + IT 10 Testcontainers (inclui `oneUserAcrossMultipleUnits` provando a promoção da PK composta no V11).
Suíte completa (run limpo) **412 testes, 0 falhas**. OpenAPI 90 paths. Sem frontend (a matriz é editada
dentro da tela de admin de usuário, ainda não migrada ao front — como os grids RBAC PRFCON/USUCON).

## Open questions
- **OQ1 — semântica dos flags:** os nomes limpos de ~5 flags ambíguos (BloqAgr=Agravo? BloqCms=Cons.Mun.Saúde? BloqImp=Impressão? BloqGra=Gráfico? PerSoaBnafar) precisam de confirmação KB/produto. A **polaridade** (Bloq=nega módulo, Per=concede) é clara pelo prefixo. Não afeta o mapeamento físico, só o nome de exibição.
- **OQ2 — SAU_USUUNI1 (setores liberados por user×unit) DIFERIDO** — segunda tabela PK composta (UsuCod,UniUsuCod,UsuSetorCod)→SAU_UNISETOR (R7/R8). Sub-slice futuro. Baseline tem stub mínimo de SAU_USUUNI1.
- **OQ3 — RBAC:** esta matriz É fonte de autorização; o `PermissionResolver` (SAU_RBAC, [[reference_rbac_engine_not_wired]]) deveria consumir estes flags p/ enforcement por (user,unit). Confirmar o papel real (SAUDE_ADMIN inferido; legado gateia via programa SAU_USU).
- **OQ4 — parity:** `/verify-parity SAU_USUUNI` antes do cutover.
