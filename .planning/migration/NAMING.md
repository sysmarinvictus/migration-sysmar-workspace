# Naming Map — GeneXus → domain

Used by `gx-extractor` and the generators to translate GeneXus Portuguese-abbreviated names into
readable domain/package/REST names. The **physical DB schema is NOT renamed** (entities keep
`@Table(name="SAU_PAC")`/`@Column(name="PacPesNom")`); clean names appear only in packages, DTOs,
endpoints, and the UI.

## Entity prefix → domain (package + REST plural)

| GeneXus prefix | Portuguese | domain (package) | REST plural |
|---|---|---|---|
| `pac` | paciente | `paciente` | `/api/pacientes` |
| `pes` | pessoa | `pessoa` | `/api/pessoas` |
| `pro` / `pesf` | profissional de saúde | `profissional` | `/api/profissionais` |
| `fun` | funcionário | `funcionario` | `/api/funcionarios` |
| `rem` | remédio / medicamento | `medicamento` | `/api/medicamentos` |
| `remlot` | lote de medicamento | `lote-medicamento` | `/api/medicamentos/lotes` |
| `remobs` | posologia | `posologia` | `/api/posologias` |
| `recesp` | receituário especial | `receituario-especial` | `/api/receituarios-especiais` |
| `esp` | especialidade | `especialidade` | `/api/especialidades` |
| `proesp` | especialidade do profissional | `profissional-especialidade` | `/api/profissionais/{id}/especialidades` |
| `uni` | unidade de atendimento | `unidade` | `/api/unidades` |
| `unisetor` | setor da unidade | `setor` | `/api/unidades/{id}/setores` |
| `usu` | usuário do sistema | `usuario` | `/api/usuarios` |
| `prf` | perfil | `perfil` | `/api/perfis` |
| `prg` | programa do sistema | `programa` | `/api/programas` |
| `bai` | bairro | `bairro` | `/api/bairros` |
| `dis` | distrito | `distrito` | `/api/distritos` |
| `loc` | local | `local` | `/api/locais` |
| `cep` | CEP | `cep` | `/api/ceps` |
| `concla` | conselho de classe | `conselho-classe` | `/api/conselhos-classe` |
| `cbor` | CBO (ocupação) | `ocupacao` | `/api/ocupacoes` |
| `tiprem` | tipo de medicamento | `tipo-medicamento` | `/api/tipos-medicamento` |
| `aprrem` | forma de apresentação (do medicamento) | `forma-apresentacao` | `/api/formas-apresentacao` |
| `imp` | impedimento | `impedimento` | `/api/impedimentos` |
| `log` | log / auditoria | `auditoria` (→ `common/audit`) | n/a |
| `emp` | empresa | `empresa` | `/api/empresas` |
| `mun` | município | `municipio` | `/api/municipios` |
| `paccns` | CNS do paciente | `paciente-cns` | `/api/pacientes/{id}/cns` |
| `pacprn` | prontuário | `prontuario` | `/api/prontuarios` |
| `par` | parâmetro | `parametro` | `/api/parametros` |

## Column convention → DTO field

GeneXus attributes are `<Prefix><Field>` (e.g. `PacPesNom`, `PacPesCPFCNPJ`, `UniCnes`). Strip the
prefix, lowercase camelCase the remainder, expand common abbreviations:

| Fragment | field name |
|---|---|
| `...Nom` | `nome` |
| `...Cod` | `codigo` (or `id` if it is the PK) |
| `...CPFCNPJ` | `cpfCnpj` |
| `...CNS` | `cns` |
| `...End` / `...EndNum` / `...EndCom` | `endereco` / `numero` / `complemento` |
| `...CEP` | `cep` |
| `...Tip` | `tipo` |
| `...Cnes` | `cnes` |
| `...Dat` / `...DtNasc` | `data` / `dataNascimento` |
| `usu_login` | `usuarioLogin` (audit actor) |

Keep a per-slice override list in the SLICE-SPEC `fields[].name` when the heuristic is wrong. The
generated entity always pins the physical name via `@Column(name="<exact GeneXus column>")`.
