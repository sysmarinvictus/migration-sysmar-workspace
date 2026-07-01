# SLICE-SPEC — SAU_PESF_PAC (Cadastro de Paciente / wizard) · Wave-6 · **COLLAPSED into SAU_PAC**

```yaml
slice: SAU_PESF_PAC
domain: paciente
description: "Cadastro de Paciente (wizard) — person(SYS_PES) + patient(SAU_PAC) composite create"
wave: 6
complexity: XL                         # sau_pesf_pac_impl.java ~746 KB (mostly the shared person-cadastro UI)
primary_table: SYS_PES                 # INSERT/UPDATE/DELETE SYS_PES + After-Trn psau_pesf_pac → INSERT SAU_PAC
status: collapsed
status_was: pending
collapsed_into: SAU_PAC                 # like SAU_PACPRN → Paciente (BACKLOG:62)
decision_date: 2026-07-01
decision_by: user (recommended, confirmed)
```

## Por que colapsado (análise 2026-07-01)
`sau_pesf_pac` é o **wizard "Cadastro de Paciente"** (título "Paciente"). Sua trilha:
`INSERT/UPDATE/DELETE SYS_PES` (pessoa) + After-Trn `psau_pesf_pac` → `INSERT SAU_PAC`. O INSERT do
`psau_pesf_pac.java` grava **exatamente as 26 colunas SAU_PAC** já mapeadas na entidade `Paciente`.

⇒ O núcleo (pessoa SYS_PES + paciente SAU_PAC, composite create) **já é entregue por `POST /api/pacientes`**
da fatia **SAU_PAC** (`PacienteService.create`). Construir um segundo endpoint de criação de paciente
duplicaria a lógica e criaria duas formas de cadastrar paciente — precedente SAU_PACPRN (dobrado em Paciente).

**A app legada tem DUAS trilhas de criação de paciente** (ambas linkadas do WorkWith `hwwsau_pac`):
- `sau_pac` — transação de manutenção do paciente (suporta INS; validação mais leve). → **migrada** (SAU_PAC slice).
- `sau_pesf_pac` — wizard de cadastro (validação mais completa + multi-subtipo). → **esta fatia**.

## Deltas do `sau_pesf_pac` sobre o já entregue (registrados como DIFERIDOS)
1. **Validação de pessoa mais completa** — o wizard exige a bateria toda do cadastro: `Informe o CEP!`,
   `Informe o bairro!`, `Informe a Nacionalidade!`, `Informe o Sexo do Paciente!`, `Informe a Data de
   Nascimento!`, além de Nome/CPF-ou-CNS/CNS/Nome Social. O `PacienteService.create` atual é
   **intencionalmente mais leve** (pacientes frequentemente têm dados parciais — isenção de CPF por
   unidade `SAU_UNI.UniCadCPF`, flag `PacSituacaoRua`/morador de rua). **DIFERIDO:** se o produto quiser
   um perfil de "cadastro estrito" de paciente, adicionar a bateria completa ao `PacienteService.create`
   (reusa a lógica de `PessoaCadastroService`); decisão de produto — hoje a leniência é proposital.
2. **Provisionamento multi-subtipo** — o After-Trn do wizard, via `tipoCadastro` (AV43TipCad ∈ {1,2,3},
   `sau_pesf_pac_impl.java:3288,3353`), pode também criar **SAU_FUN (funcionário) / SAU_PRO (profissional)**
   além do paciente (`psau_pesf_fun`/`psau_pesf_pro`). **FORA DE ESCOPO** — mesmo padrão de multi-subtipo
   do SAU_PESF, deliberadamente adiado (SAU_PESF OQ1 = "cadastro puro"; cross-entity writes numa fatia de
   pessoa). Se necessário no futuro, é um fluxo próprio (ex.: `POST /api/pessoas/{id}/subtipos`).

## Nada gerado (sem código)
Fatia colapsada — nenhum endpoint/entidade novo. Cobertura efetiva:
- Criar paciente (pessoa+SAU_PAC): `POST /api/pacientes` (SAU_PAC). ✓
- Editar/excluir paciente: `PUT/DELETE /api/pacientes/{id}` (SAU_PAC). ✓
- Os dois deltas acima ficam como itens de backlog (refinamento de validação; multi-subtipo).
