# SLICE-SPEC — SAU_PACCNS (CNS do Paciente) · Wave-6 · **COLLAPSED into SAU_PAC**

```yaml
slice: SAU_PACCNS
domain: paciente
description: "Editor do CNS (Cartão Nacional de Saúde) do paciente — um único escalar em SYS_PES.PesNumCns"
wave: 6                                 # BACKLOG lista em Wave 5 (diferido); re-planejado em Wave 6 como comportamento do SAU_PAC
complexity: M                          # sau_paccns_impl.java ~104 KB (a maior parte é a UI de cadastro compartilhada)
primary_table: SYS_PES                 # NÃO existe tabela SAU_PACCNS — ver análise
status: collapsed
status_was: pending
collapsed_into: SAU_PAC                 # precedente: SAU_PACPRN → Paciente (BACKLOG:62), SAU_PESF_PAC → SAU_PAC (BACKLOG:105)
decision_date: 2026-07-01
decision_by: evidence (live-DB introspection + GeneXus spec) — precedente SAU_PACPRN/SAU_PESF_PAC
depends_on: [SYS_PES, SAU_PAC]          # ambos já migrados
```

## Por que colapsado (análise 2026-07-01 — dois agentes convergiram independentemente)

**`SAU_PACCNS` não é uma tabela e não é um histórico multi-CNS.** Confirmado por três fontes:

1. **Introspecção do banco vivo** (`host.docker.internal:5432`, db `saude-mandaguari`): **não existe** nenhuma
   relação `sau_paccns` (nem variação de caixa/aspas) e **não existe** coluna `pacpesnumcns`. O CNS é **um único
   escalar** `SYS_PES.pesnumcns char(20)`, uma linha por pessoa (PK `pescod`; 82.951/83.105 preenchidos).
2. **GeneXus spec** `Receituario/GXSPC002/GEN12/SAU_PACCNS.sp0`: `struct_i(3,0,['Context','Mode','Isauthorized','Pacpescod'])`,
   `level_i(...'ISAU_PAC'...)` — a estrutura da transação **é a base SAU_PAC**, chaveada por atributo 3 = `Pacpescod`.
   `Flagcns`, `Pescod_outro`, `Cnsduplicado` são work-variables do formulário, **não** colunas.
3. **GeneXus impl** `Receituario/JavaModel/web/sau_paccns_impl.java`: lê/grava exatamente `A146PacPesNumCns`
   (atributo 146 → físico `PesNumCns`) chaveado por `A3PacPesCod` (cursores em :779/:1226/:1333, `new Long(A3PacPesCod)`);
   auth via `pisauthorized(..., "SAU_PAC", ...)` (:692). É um **editor de campo único** do CNS na linha SYS_PES do
   paciente — não uma coleção-filha.

⇒ Toda a função da transação `sau_paccns` (ler o CNS do paciente, validar formato, garantir unicidade entre
pacientes, gravar em `SYS_PES.PesNumCns`) **já é entregue pela fatia SAU_PAC** — `POST/PUT /api/pacientes`
(`PacienteService.create`/`update`), sobre o **mesmo** mapeamento físico `Pessoa.cns → @Column("PesNumCns") char(20)`.

### Cobertura já entregue (verificada em código, SAU_PAC slice)
| Regra legada (sau_paccns) | Onde já está no SAU_PAC | Fonte |
|---|---|---|
| Gravar CNS em `SYS_PES.PesNumCns` (char 20, trim) | `Pessoa.setCns` / `Paciente` write-back | `pessoa/domain/Pessoa.java:75,330`; `paciente/.../PacienteService.java:185` |
| Validar formato do CNS (`psau_val_cns`) | R4 `CnsValidator.isValidCns` → `"CNS inválido! Tente novamente."` | `PacienteService.java:159` |
| Unicidade entre pacientes (`psau_ver_pac`) → msg *"Este número de CNS está sendo utilizado pelo paciente N!"* | R3 `findCnsOwnerAmongPatients(cns, selfId)` → `Conflict 409` (mesma mensagem, com exclusão do próprio) | `PacienteService.java:160-162` |
| CPF-ou-CNS obrigatório (isenção `SAU_UNI.UniCadCPF`) | R2 | `PacienteService.java:148` |
| Auth prefixo `"SAU_PAC"` | `hasRole('SAUDE_CADASTRO')` no `PacienteController` | precedente da fatia |

Criar um `PacienteCns @Table("SAU_PACCNS")` exigiria uma tabela **inexistente** → `hibernate.ddl-auto=validate`
falharia, e criar um stub fabricaria schema (proibido por CLAUDE.md Regra 2, "no schema redesign in Phase 1").
Não há PK composta (chave única é o escalar `PacPesCod`), então o precedente `@IdClass` **não** se aplica.

## Nada gerado (sem código, sem migração)
- **Sem entidade nova.** CNS já mapeado em `Pessoa.cns` (`PesNumCns` char(20), `@JdbcTypeCode(CHAR)`, trim-on-read).
- **Sem V14.** A coluna já existe no baseline SYS_PES; o tipo vivo (`character(20)`) já casa com o mapeamento — nenhum
  `ALTER ... TYPE` de alinhamento necessário (diferente do V13 `PacProNum`→CHAR(10)).
- Editar/criar o CNS do paciente: `POST/PUT /api/pacientes` (SAU_PAC). ✓ Toda leitura/escrita já é auditada (PHI/LGPD).

## Deltas do `sau_paccns` sobre o já entregue (registrados como DIFERIDOS)
1. **Endpoint dedicado de edição só-do-CNS** — a tela legada é um editor de campo único. O SAU_PAC já permite editar o
   CNS via `PUT /api/pacientes/{id}` (update completo). Um `PUT /api/pacientes/{pacPesCod}/cns` isolado seria mera
   conveniência de UI e criaria **um segundo caminho de escrita** para a coluna compartilhada `SYS_PES.PesNumCns`
   (risco de dono-duplo / optimistic-lock que o gx-schema-mapper sinalizou). **DIFERIDO** — só implementar se o produto
   exigir a tela isolada; reusaria `PacienteService` (sem nova regra). Nenhuma lógica de negócio nova.
2. **Comprimento CNS: coluna 20 vs. validação 15 dígitos** — `psau_val_cns` (`impl:875`) rejeita CNS ≠ 15 dígitos,
   enquanto a coluna física é `char(20)`. O SAU_PAC já valida via `CnsValidator` (formato/dígito verificador) mantendo a
   coluna char(20). Se surgir divergência de parity no tamanho exato, ajustar `CnsValidator`; hoje considerado coberto.

## Open questions (para o dono da KB — não bloqueiam o colapso)
- Confirmar que **não** há requisito real de multi-CNS-por-paciente (nada no DB/código o suporta hoje). Se um dia
  existir, aí sim vira uma coleção-filha própria (`GET/POST/DELETE /api/pacientes/{id}/cns/...`) com tabela nova.
- Se o produto quiser paridade 1:1 com a tela isolada de CNS, promover o Delta 1 a item de backlog.
