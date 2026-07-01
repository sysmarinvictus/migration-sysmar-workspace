# SLICE-SPEC — SAU_PESF_PROFEXT (Cadastro de Profissional Externo) · Wave-4 · composite create

> Registers an **external professional**: a LEAN composite create of a person (SYS_PES, PesTip=1) **and**
> an external professional (SAU_PRO, ProExt=1), in one call. Composes two already-migrated entities
> (`pessoa/` write path from SAU_PESF, `profissional/` from SAU_PRO). Grounded 2026-07-01 (impl grep +
> psau_pesf_pro). **Closes Wave-4.**

```yaml
slice: SAU_PESF_PROFEXT
domain: profissionalexterno         # /api/profissionais-externos
description: Cadastro de Profissional Externo
wave: 4
complexity: M                        # sau_pesf_profext_impl.java ~173 KB
primary_table: SYS_PES               # INSERT/UPDATE/DELETE SYS_PES (lean) + After-Trn INSERT SAU_PRO
impl: sau_pesf_profext_impl.java     # + psau_pesf_pro.java (SAU_PRO insert), psau_val_cns, psau_ver_cns
depends_on:
  - { table: SYS_PES,  migrated: true,  note: "person write path (SAU_PESF / pessoa/)" }
  - { table: SAU_PRO,  migrated: true,  note: "professional (profissional/)" }
  - { table: SYS_MUN,  migrated: "?",   col: PesMunCod, role: FK-lookup (required, R18/R19) }
  - { table: SAU_CONCLA, migrated: true, col: ConClaCod, role: FK-lookup (required, R21) }
referenced_by: []   # delete of the person is guarded by SAU_PRO/FUN/PAC (same triad as SAU_PESF)

# ── LEAN form (7 controls): Nome, CNS, Município, Conselho de Classe, Nº do Conselho (ProNumCR),
#    Data de Cadastro (default today). NO CPF/RG/endereço/sexo/cor/nacionalidade (hardcoded '' in legacy).
fields:
  - { gx: PesNom,      name: nome,             required: true,  note: "R3-R10 name rules; R12 stored UPPERCASE" }
  - { gx: PesNumCns,   name: cns,              required: true,  note: "R13 req, R14-R16 valid, R17 unique among professionals" }
  - { gx: PesMunCod,   name: municipioCod,     required: true,  note: "R18 req + R19 exists (SYS_MUN)" }
  - { gx: ConClaCod,   name: conselhoClasseCod, required: true, note: "R21 req (+ exists — safe improvement)" }
  - { gx: ProNumCR,    name: numeroConselho,   required: true,  note: "R20 req" }
  - { gx: ProDatFim,   name: dataFim,          required: false }
# SAU_PRO provisioning (psau_pesf_pro): ProExt=1 (R29), ProSit=1, ProDatIni=today, cert fields '' (R30).

endpoints:
  - { method: POST, path: /api/profissionais-externos, note: "composite create (person + SAU_PRO externo)" }
  - { method: GET,  path: /api/profissionais-externos/{id}, note: "read-back" }
  # edit/delete delegate to the profissional (SAU_PRO) flow — mirrors the legacy post-create redirect.
phi_fields: [nome, cns, municipioCod]      # person cadastro (audited)
auth: { roles_required: [SAUDE_CADASTRO] }
status: tested
status_was: pending
```

## Regras (citação — sau_pesf_profext_impl.java / psau_pesf_pro.java)
- **R3-R10** nome (obrigatório :1179; ≥3 :1186; sobrenome :1193; sem espaço duplo :1200; só letras :1207; +R8/R9/R10 micro-regras). **Reusa `PessoaNomeValidator` (R3-R7); R8/R9/R10 DIFERIDAS** como em SAU_PESF.
- **R12** nome gravado MAIÚSCULO (:657) — aplicado.
- **R13** CNS obrigatório (:1242); **R14-R16** valida via `CnsValidator` (tam 15 / dígitos / DV, psau_val_cns); **R17** CNS único entre profissionais (INS-only, psau_ver_cns, escopo PesTip=1) → `profissionalRepo.findCnsOwner` → 409.
- **R18** município obrigatório (:1315); **R19** existe em SYS_MUN (:1296) → `pessoaRepo.municipioExists`.
- **R20** nº do conselho obrigatório (:1322); **R21** conselho obrigatório (:1329) + **exists** (`conselhoRepo.existsById` — melhoria segura vs. lookup soft do legado R22, que inseriria FK pendente; dado real vem do picker).
- **R23** PesCadDat default hoje; **R24** PesTip=1; **R25** PesCod=MAX+1 (psau_inc_pes); **R26** soundex (`SoundexService`).
- **R27** insert SYS_PES enxuto (demais colunas ''/0); **R28** After-Trn insert SAU_PRO (psau_pesf_pro:232); **R29** ProExt=1, ProSit=1; **R30** sem certificado (cert fields null).
- **R31-R33** delete-guards SYS_PES (SAU_PRO/FUN/PAC) — já implementados/testados em `PessoaCadastroService` (delete via fluxo pessoa/profissional).
- **R34** auditoria (psau_inc_log) → `common/audit` CREATE/READ.

## Decisões / notas
- **Atomicidade (melhoria intencional):** o legado commita SYS_PES e depois SAU_PRO em transações SEPARADAS (falha no SAU_PRO deixa pessoa órfã). Aqui os dois writes são UM `@Transactional` (tudo-ou-nada). Registrado.
- **Edit/delete** do profissional externo vão pelo fluxo `profissional/` (SAU_PRO) — espelha o redirect pós-create do legado; este slice expõe só POST + GET.
- **Sem CPF** nesta transação (o legado não coleta/valida CPF aqui; PesCPFCNPJ=''). 
- Reusa: `pessoa/` (Pessoa entity, PessoaRepository.findMaxPesCod/municipioExists/delete-guards, PessoaNomeValidator agora public), `profissional/` (Profissional entity, ProfissionalRepository.findCnsOwner), `SoundexService`, `CnsValidator`, `conselhoclasse/` (ConselhoClasseRepository).

## Resultado (2026-07-01 — /migrate-slice)
Backend (profissionalexterno/: service composto + dto + controller) + testes (Service 12, IT 8
Testcontainers — verifica que AMBAS SYS_PES[PesTip=1] e SAU_PRO[ProExt=1] são criadas). Suíte completa
**485 testes, 0 falhas**. Frontend: form enxuto + rota/nav. OpenAPI 88 paths. Fecha a Wave-4.

## Open questions
- **OQ1 — R2 RBAC:** `pisauthorized("SAU_PESF_PROFEXT")` não plugado (SAU_RBAC existe, não wired) — [[reference_rbac_engine_not_wired]]. SAUDE_CADASTRO por ora.
- **OQ2 — R16 algoritmo CNS** e **R11 psau_limpacaracter** (normalização de nome) não portados exatamente — verificar na parity.
- **OQ3 — parity:** `/verify-parity SAU_PESF_PROFEXT` antes do cutover (cria SYS_PES+SAU_PRO — comparar as duas linhas no golden-master).
