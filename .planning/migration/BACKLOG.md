# Migration Backlog — dependency-ordered waves

**Generated:** 2026-06-01 (by `gx-inventory` analysis) · **Transactions:** 34 + 2 BC-only (`sys_pes`, `sau_remlot`)

Migrate **top wave first**. Within a wave, transactions are independent of each other. The
**authoritative `depends_on` list for each slice is computed at extract time** from its
`Metadata/TableAccess/<Trn>.xml`; the waves below encode the verified ordering skeleton
(foundation → leaves → cycles → patient/prescription core).

Complexity = `_impl.java` size. Status starts `pending`; updated by `migration-status`.

## Summary

| Wave | Theme | Transactions | Start here? |
|------|-------|--------------|-------------|
| 0 | Security / audit / person foundation (cross-cutting) | 11 | after scaffold |
| 1 | Pure leaf lookups (no migratable deps) | 6 | **first real slices** |
| 2 | Geography + units + specialty (some cycles) | 5 | |
| 3 | Medication catalog | 4 | |
| 4 | Professionals | 4 | |
| 5 | Parameters + patient links | 4 | |
| 6 | Patient + special prescription (core) | 2 | last |

**Recommended warm-up slice:** `sau_concla` (Conselho de Classe, S, 97 KB) or `sau_loc` (Local, S) —
smallest full constellation, proves the factory end-to-end fast. Then the chosen reference slice
**`sau_pac` (Wave 6, XL)** exercises the hard path.

---

## Wave 0 — Security / audit / person foundation

These tables (`SAU_USU`, `SAU_USUCON`, `SAU_PRF`, `SAU_PRFCON`, `SAU_LOG`) appear in nearly every
TableAccess XML — GeneXus injects audit + per-profile permission checks everywhere. They form a
**cyclic cluster**; migrate as one coordinated group alongside the JWT/Spring-Security baseline.
`SYS_PES` (person supertype, BC-only) and `SYS_EMP` underpin every personal entity.

| Transaction | Primary table | Description | Complexity | Notes |
|---|---|---|---|---|
| `sys_pes` (BC) | SYS_PES | Pessoa (supertype) | M (no UI) | base of pac/pro/fun/usu |
| `sys_emp` | SYS_EMP | Empresa | M | |
| `sau_usu` | SAU_USU | Usuário do sistema | XL (757) | drives auth slice |
| `sau_usucon` | SAU_USUCON | Controle de Acesso | M | |
| `sau_usupar` | SAU_USUPAR | Usuário — parâmetros | M | |
| `sau_prf` | SAU_PRF | Perfil | M | |
| `sau_prfcon` | SAU_PRFCON | Controle de acesso por perfil | M | |
| `sau_prg` | SAU_PRG | Programas do Sistema | M | |
| `sau_prggrp` | SAU_PRGGRP | Grupos de Programas | S | |
| `sau_log` (BC) | SAU_LOG | Log do Sistema (auditoria) | M | → `common/audit` module |
| `sau_fun` | SAU_FUN | Funcionário / Função | M (286) | |

## Wave 1 — Pure leaf lookups (start here)

Own one small table, depend only on the Wave-0 foundation.

| Transaction | Primary table | Description | Complexity |
|---|---|---|---|
| `sau_concla` | SAU_CONCLA | Conselho de Classe | S (97) |
| `sau_loc` | SAU_LOC | Local | S (102) |
| `sau_remobs` | SAU_REMOBS | Posologia | M (120) |
| `sau_bai` | SAU_BAI | Bairro | S (87) |
| `sau_tiprem` | SAU_TIPREM | Tipos de Medicamento | S (84) |
| ~~`sau_pacprn`~~ | ~~SAU_PACPRN~~ | ~~Prontuário do paciente~~ | ~~S (78)~~ | **COLLAPSED into SAU_PAC** — no physical table; `PacProNum` field folded into Paciente entity (Wave 6). See `slices/SAU_PACPRN.slice.md`. |

## Wave 2 — Geography, units, specialty

| Transaction | Primary table | Description | Complexity | Cycle |
|---|---|---|---|---|
| `sau_dis` | SAU_DIS | Distrito | M (158) | bai↔dis |
| `sau_uni` | SAU_UNI | Unidade de Atendimento | XL (1056) | uni↔unisetor |
| `sau_unisetor` | SAU_UNISETOR | Setor da unidade | M (152) | uni↔unisetor |
| `sau_esp` | SAU_ESP | Especialidades | M (226) | pro/proesp/esp cluster |
| `sau_imp` | SAU_IMP | Impedimento | M (155) | |

## Wave 3 — Medication catalog

| Transaction | Primary table | Description | Complexity | Cycle |
|---|---|---|---|---|
| `sau_rem` | SAU_REM | Medicamento | L (691) | rem↔aprrem↔tiprem |
| `sau_aprrem` | SAU_APRREM | Aprovação de medicamento | S (89) | rem↔aprrem↔tiprem |
| `sau_remlot` (BC) | SAU_REMLOT | Lote de medicamento | — | |
| `sau_renameatualizado` | SAU_RENAME... | RENAME atualizado | M (116) | confirm meaning in KB |

## Wave 4 — Professionals

| Transaction | Primary table | Description | Complexity | Cycle |
|---|---|---|---|---|
| `sau_pro` | SAU_PRO | Profissionais | L (538) | pro↔proesp↔esp↔usuuni↔imp |
| `sau_proesp` | SAU_PROESP | Especialidade do Profissional | L (308) | pro↔proesp↔esp |
| `sau_pesf` | SAU_PESF | Pessoa Física (prof. de saúde) | L (598) | |
| `sau_pesf_profext` | SAU_PESF... | Cadastro de Profissional Externo | M (169) | |

## Wave 5 — Parameters + patient links

| Transaction | Primary table | Description | Complexity |
|---|---|---|---|
| `sau_par_amb` | SAU_PAR_AMB | Parâmetro ambulatorial | L (582) |
| `sau_par_ger` | SAU_PAR_GER | Parâmetro geral / Gerente | M (138) |
| ~~`sau_paccns`~~ | ~~SAU_PACCNS~~ | ~~CNS do paciente~~ | ~~M (104)~~ | **COLLAPSED into SAU_PAC** (2026-07-01) — no physical table; the CNS is one scalar `SYS_PES.PesNumCns` already written+validated (R2/R3/R4) by `POST/PUT /api/pacientes`. See `slices/SAU_PACCNS.slice.md`. |
| `sau_usuuni` | SAU_USUUNI | Usuário × Unidade | L (509) |

## Wave 6 — Patient + special prescription (core, last)

| Transaction | Primary table | Description | Complexity | Cycle |
|---|---|---|---|---|
| `sau_pesf_pac` | SAU_PESF_PAC | Cadastro de Paciente (wizard) — **COLLAPSED into SAU_PAC** (2026-07-01); core=`POST /api/pacientes`, deltas (fuller validation, multi-subtype) deferred | XL (728) | |
| `sau_pac` | SAU_PAC | **Paciente** (reference slice) — **tested** | XL (925) | pac↔recesp |
| `sau_recesp` | SAU_RECESP | Receituário Controle Especial | L (460) | pac↔recesp · **Portaria 344/98** |

---

## Dependency cycles (break manually)

| Cycle | How to break |
|---|---|
| `bai ↔ dis` | Migrate both in Wave 2; create FKs as nullable, backfill after both exist. |
| `uni ↔ unisetor` | Same wave; unisetor is a child of uni — migrate uni entity first, then unisetor. |
| `rem ↔ aprrem ↔ tiprem` | tiprem (W1) first, then rem, then aprrem; soft FK during transition. |
| `pro ↔ proesp ↔ esp ↔ usuuni ↔ imp` | esp (W2) + imp (W2) first; pro then proesp; usuuni (W5). |
| `pac ↔ recesp` | pac entity first (read side), recesp references it; recesp INSERT validated last. |

## Notes / open questions
- Blank descriptions in headers are GeneXus mojibake; confirm Portuguese names against the KB.
- `sau_renameatualizado` purpose unclear — verify in GeneXus IDE before specing.
- `SAU_USU` password hashing scheme is the gating unknown for the JWT auth slice (Wave 0).
- `SAU_RECESP` must be validated against Portaria SVS/MS 344/98 with a CRF/regulatory lead.
