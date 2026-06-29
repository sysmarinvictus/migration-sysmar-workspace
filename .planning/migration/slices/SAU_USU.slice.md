# SLICE-SPEC — SAU_USU (Usuário do Sistema) · **FATIA DE AUTENTICAÇÃO**

> **Wave-0, keystone de segurança.** Substitui o login-stub de desenvolvimento por autenticação real
> SAU_USU + RBAC (JWT), e resolve o `logusucod` real do SAU_LOG e o auth por-modo do SAU_PRO (R28).
> ⚠️ **ACHADO DE SEGURANÇA GRAVE:** as senhas estão em **criptografia REVERSÍVEL** (`encrypt64(texto, usukey)`,
> chave por-usuário em `usukey`) — **não é hash**. 1228/1230 usuários, 24 chars base64. Além disso há
> **dois backdoors de superusuário SYSMAR**. Esta fatia exige **sign-off de segurança/regulatório ANTES
> de `/migrate-slice`** (a estratégia de migração de senha é a decisão que destrava tudo). Extraído 2026-06-28.

```yaml
slice: SAU_USU
domain: usuario                   # pacote …/seguranca (auth) — replaces dev-stub login
description: "Usuários do Sistema"
wave: 0
complexity: XL                    # sau_usu_impl.java 775 KB; 110 colunas; 18 regras
primary_table: SAU_USU            # PK usucod (int), 1230 linhas
role: "authentication/authorization keystone — JWT + RBAC; resolve SAU_LOG.logusucod e SAU_PRO R28"
constellation:
  transaction: sau_usu.java
  impl:        sau_usu_impl.java          # 775 KB — CRUD de usuário + validações + audit
  bc:          null                       # não-BC → introspecção do banco vivo é autoritativa
  workwith:    hwwsau_usu_impl.java        # /api/usuarios (lista)
  view:        hviewsau_usu_impl.java      # /api/usuarios/{id}
  prompt:      hpromptsau_usu_impl.java     # /api/usuarios/lookup
  sections:    [hsau_usugeneral_impl.java, hsau_usu_exc_impl.java]
  auth_flow:                              # superfície de segurança primária:
    login_form:   hindex_impl.java          # form de login + verificação de senha (S122) + SETCONTEXT (S135)
    login_access: acessamodulo.java          # gate de MÓDULO + bypass SYSMAR (:79)
    login_lookup: buscaloginusuario.java      # resolve usuário por login
    change_pw:    hmudasenhalogin_impl.java   # troca de senha (decrypt-then-compare :673-674)
    change_proc:  psau_usu_mudasenha.java     # encrypt-before-store + rotação de chave (:71-80,162)
    authorize:    pisauthorized.java           # RBAC por-modo Inc/Alt/Exc/Con (:69-237, cursores :371-374)
    load_context: ploadcontext.java            # carrega contexto de sessão
    verify_login: psau_verif_usu.java          # unicidade de login

depends_on:                       # SÓ SAU_PRO está migrado; o resto é Wave-0/4 não migrado.
  - { table: SAU_PRO,    role: "profissional (UsuProPesCod)", migrated: true }
  - { table: SAU_PRF,    role: "perfil/role (UsuPrfCod) — tier primário do RBAC", migrated: false, wave: 0 }
  - { table: SAU_PRFCON, role: "perm. por perfil (PrfPrgInc/Alt/Exc/Con)", migrated: false, wave: 0 }
  - { table: SAU_USUCON, role: "perm. por usuário (UsuInc/Alt/Exc/Con)", migrated: false, wave: 0 }
  - { table: SAU_PRG,    role: "programas (PrgCod = espaço de permissões)", migrated: false, wave: 0 }
  - { table: SAU_USUUNI, role: "usuário×unidade — login é por (login, unidade); gates de módulo", migrated: false, wave: 4 }
  - { table: SAU_FUN,    role: "funcionário (FunPesCod)", migrated: false }
  - { table: SYS_PES/SAU_UNI, role: "pessoa/unidade (contexto)", migrated: false }
# MITIGAÇÃO: para o login + autoridades, introspectar SAU_PRF/USUCON/PRFCON/USUUNI READ-ONLY (sem migrar
# suas telas CRUD ainda). Login real precisa ler essas tabelas para montar as GrantedAuthorities.

# ── ENTIDADE: mapear só o subconjunto AUTH-ESSENCIAL (16 cols); deixar ~94 flags NÃO mapeadas ──
# (ddl-auto=validate só confere as colunas MAPEADAS — entidade focada valida contra a tabela de 110 cols)
fields:
  - { gx: UsuCod,        name: usuCod,        type: int,    pk: true, nullable: false }
  - { gx: UsuNom,        name: nome,          type: string, len: 50,  nullable: true, phi: true }
  - { gx: UsuLogin,      name: login,         type: string, len: 20,  nullable: true,
      note: "chave de login. NÃO há UNIQUE no banco (só índice não-único usau_usu) → unicidade no serviço (R13)" }
  - { gx: UsuSen,        name: senha,         type: string, len: 100, nullable: true, secret: true,
      note: "⚠ @JsonIgnore. CRIPTOGRAFIA REVERSÍVEL encrypt64(texto, usukey) — NÃO hash. Nunca retornar/logar." }
  - { gx: UsuKey,        name: chaveSenha,    type: string, len: 100, nullable: true, secret: true,
      note: "@JsonIgnore. Chave simétrica por-usuário (0/32 chars), rotacionada a cada troca de senha (R11)." }
  - { gx: UsuTip,        name: tipo,          type: short,  nullable: true, jdbc: SMALLINT }
  - { gx: UsuBloq,       name: bloqueado,     type: short,  nullable: true, jdbc: SMALLINT, note: "1=bloqueado (R5)" }
  - { gx: UsuPrfCod,     name: perfilId,      type: int,    nullable: true, fk: SAU_PRF, note: "raw id (perfil não migrado)" }
  - { gx: UsuSysmar,     name: superusuario,  type: boolean, nullable: true, note: "⚠ bypass de superusuário (R2)" }
  - { gx: UsuProPesCod,  name: profissionalId, type: long,  nullable: true, fk: SAU_PRO }
  - { gx: FunPesCod,     name: funcionarioId, type: long,   nullable: true, fk: SAU_FUN }
  - { gx: UsuTokenSoa,   name: tokenSoa,      type: string, len: 5000, nullable: true, note: "propósito incerto (OQ6)" }
  - { gx: UsuTokenExp,   name: tokenExpiracao, type: int,   nullable: true }
  - { gx: UsuTokenData,  name: tokenData,     type: timestamp, nullable: true }
  - { gx: UsuDataUltimoAcesso, name: ultimoAcesso, type: date, nullable: true }
  - { gx: UsuDataRedefinicao,  name: dataRedefinicaoSenha, type: date, nullable: true, note: "carimbo de troca de senha (R11)" }
fields_unmapped:   # presentes na tabela, NÃO mapeadas na entidade v1 (validate ignora):
  - "~55 usubloq* (smallint) + 3 booleanas (usubloqprtcon/prtodo/resexa) — bitmap de acesso a módulos"
  - "16 usupesquisa* (boolean) — permissões de BUSCA de paciente (LGPD; default negado, R15)"
  - "usuunicod, usubio(text), e ~20 toggles de comportamento — fora do escopo da auth v1"
  # Quando uma fatia de autorização precisar → @Embeddable PermissoesPesquisaPaciente / PermissoesModulo.

keys:
  pk: [UsuCod]
  unique: []                      # NENHUM unique físico (só índice não-único usau_usu em usulogin)
  indexes: [{ index: usau_usu, cols: [UsuLogin], unique: false }, { index: isau_usu3, cols: [UsuUniCod] }]
  fks: []                         # ZERO FKs físicas (RI no app); ids escalares, sem @ManyToOne

endpoints:
  # PÚBLICO (não autenticado):
  - { method: POST, path: /auth/login,           auth: PUBLIC, note: "substitui o stub; verifica senha (esquema legado), checa bloqueio (R5), emite JWT access+refresh" }
  - { method: POST, path: /auth/refresh,         auth: PUBLIC }
  - { method: POST, path: /auth/change-password, auth: "self (autenticado)", from: hmudasenhalogin, note: "exige senha atual; rotaciona chave; carimba dataRedefinicao (R10/R11)" }
  # ADMIN (CRUD de usuários):
  - { method: GET,    path: /api/usuarios,        auth: ADMIN, from: hwwsau_usu }
  - { method: GET,    path: /api/usuarios/{id},   auth: ADMIN }
  - { method: POST,   path: /api/usuarios,        auth: ADMIN, note: "login req+único (R12/R13); FK perfil/prof (R14); usupesquisa* default negado (R15)" }
  - { method: PUT,    path: /api/usuarios/{id},   auth: ADMIN, note: "bloquear/desbloquear via UsuBloq (R16)" }
  - { method: DELETE, path: /api/usuarios/{id},   auth: ADMIN, note: "bloqueado se referenciado por SAU_USUCON/SAU_USUUNI (R17)" }
  - { method: GET,    path: /api/usuarios/lookup, auth: authenticated }

phi_fields: [nome]   # UsuNom (PII). usupesquisa* (não mapeadas v1) são CONTROLE-DE-ACESSO a PHI (LGPD).
# senha/chaveSenha (UsuSen/UsuKey) são SEGREDOS, não PHI — @JsonIgnore, nunca logar/retornar.

auth:
  login_public: true
  superuser_bypass: "UsuSysmar=true (pisauthorized) E login=='SYSMAR' (acessamodulo) — DOIS caminhos (R2)"
  rbac_model: >
    Two-tier por-programa (pisauthorized): se UsuSysmar→tudo; senão se o usuário TEM perfil válido
    (UsuPrfCod→SAU_PRF) as permissões vêm de SAU_PRFCON (PrfPrgInc/Alt/Exc/Con por PrgCod); senão de
    SAU_USUCON por-usuário (UsuInc/Alt/Exc/Con). Perfil tem precedência (NÃO é OR). Gate de módulo
    adicional: acessamodulo lê flags UsuUniBloq* de SAU_USUUNI por (UsuCod,UniCod). (R8/R9)
  roles_required_admin: [SAUDE_ADMIN]   # INFERIDO — confirmar (OQ8)
  source: "pisauthorized.java:69-237,371-374; acessamodulo.java:64-413"

parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment (login form hindex)"
  scenarios:
    - "login válido (login+senha corretos) → autentica (verificação = decrypt64(ususen,usukey)==senha) + emite token"
    - "login inválido / senha errada → MESMA mensagem genérica (sem enumeração de usuário) (R3)"
    - "usuário bloqueado (UsuBloq=1) → rejeitado APÓS senha correta (R5)"
    - "troca de senha: exige senha atual correta + confirmação; rotaciona usukey; carimba dataRedefinicao (R10/R11)"
    - "autorização: perfil tem precedência sobre per-usuário; bits Inc/Alt/Exc/Con por modo (R8)"
    - "SYSMAR → bypassa tudo (R2)"
    - "criar usuário: login único (R13), nome+login obrigatórios (R12), usupesquisa* default negado (R15)"
    - "deletar usuário referenciado por SAU_USUCON/SAU_USUUNI → bloqueado (R17)"

status: backend   # backend de auth gerado & compilando 2026-06-29; suite completa 542 testes 0F/0E (sem regressão).
# Gerado (…/seguranca/): Usuario (16 cols, ususen/usukey @JsonIgnore), UsuarioRepository (findByLogin, SAU_PRF
# read-only, guards de delete), SauUsuUserDetailsService (@Profile !test&!local — auth real; bcrypt p/ usukey NULL,
# usukey presente⇒"redefina senha"; UsuBloq⇒locked; SYSMAR⇒ROLE_SUPERUSER+admin+cadastro; perfil→roles coarse;
# stamp últimoAcesso + audit LOGIN), UsuarioService+Controller (/api/usuarios admin, R12-R17), ChangePasswordController
# (/auth/change-password R10/R11), V1 SAU_USU completado (16 cols). DevUserDetailsService mantido em local/test (ITs OK).
# PENDENTE (sessão nova): test-author (testes de auth) + migration-reviewer + bridge --commit no cutover.
# Deferidos: RBAC fino por-programa (SAU_PRFCON/USUCON), usupesquisa* (LGPD), validação dura de FK (R14 soft).
status_was: specced
```

## Regras mineradas (auth — citação à linha; confiança anotada)

### 🔴 Segurança-crítica (sign-off obrigatório)
- **R1** (login, authorization, **high**) — **Senha = criptografia REVERSÍVEL, não hash.** No login, `ususen` é DECRIPTADO com `Encryption.decrypt64(ususen, usukey)` e o texto comparado (trim, case-sensitive) à senha digitada. `usukey` é chave simétrica **por-usuário** (re-gerada a cada troca, R11), armazenada junto do ciphertext. Os 24-char base64 são decriptáveis dado `usukey`. src `hindex_impl.java:888-889,2415, hmudasenhalogin_impl.java:673-674`. → `AuthServiceTest#verifyPassword_decryptsStoredCipherWithPerUserKeyAndComparesPlaintext`.
- **R2** (login, authorization, **high**) — **Backdoor SYSMAR (dois caminhos):** (a) `pisauthorized` libera TODA operação se `Context.Ususysmar`==true; (b) `acessamodulo` libera TODO módulo se `login=="SYSMAR"`. Identificar quem tem esses privilégios antes do cutover. src `pisauthorized.java:65-68, acessamodulo.java:79`.

### Fluxo de login
- **R3** (validation, high) — Lookup por (UsuLogin **uppercased**, Unidade) via JOIN SAU_USUUNI. Sem match OU senha errada → **mesma mensagem genérica** "Usuário ou senha incorreta!" (sem enumeração). Login exige seleção de unidade + usuário atribuído à unidade. src `hindex_impl.java:866-918,670`.
- **R4** (validation, high) — Pré-auth: login/senha obrigatórios; gate de data do servidor; gate de licença (`psau_libvfc`). Portar (b)/(licença) como opcional/medium. src `hindex_impl.java:786-830`.
- **R5** (authorization, high) — Gate de bloqueio: APÓS senha correta, se `UsuBloq==1` → "Usuário Bloqueado!". src `hindex_impl.java:891-894`.
- **R6** (side-effect, high) — Sucesso → monta CONTEXTO de sessão (UsuCod, UsuLogin, UsuTip, Unidade, ProPesCod, EspCod, FunPesCod, Ususysmar, flags) salvo como XML na sessão; lido por `ploadcontext`. src `hindex_impl.java:1142-1248, ploadcontext.java:50-54`.
- **R7** (side-effect, **medium**) — Registra último acesso (`psau_bscdatultaces`). **NÃO há contador de tentativas, lockout, nem expiração de senha inline** (ParInaUsuDias/ParSenUsuDias são carregados mas não bloqueiam no login). Expiração/inatividade provavelmente em `halertadataexpiracao`/`desbloqueiausuario` (não minerados). src `hindex_impl.java:780-784,854-860`.

### Autorização
- **R8** (authorization, high) — Precedência por (objeto, modo): (1) Ususysmar→ok; senão (2) se perfil válido (UsuPrfCod>0 ∈ SAU_PRF) → SAU_PRFCON (PrfPrgInc/Alt/Exc/Con); senão (3) SAU_USUCON por-usuário (UsuInc/Alt/Exc/Con). ==1 libera; sem linha→nega. **Perfil tem precedência (não é OR).** src `pisauthorized.java:69-237,371-374`.
- **R9** (authorization, high) — Gate de MÓDULO (independente do CRUD): `acessamodulo` mapeia código de módulo → flag `UsuUniBloq*` em SAU_USUUNI por (UsuCod,UniCod); flag setada → negado. SYSMAR bypassa. src `acessamodulo.java:64-413`.

### Troca de senha
- **R10** (validation, high) — Validações: login/senha-atual/nova/confirmação obrigatórias; nova==confirmação; senha atual verificada por decrypt-then-compare (R1). **SEM regras de complexidade/tamanho/reuso.** src `hmudasenhalogin_impl.java:598-700`.
- **R11** (side-effect, high) — Encrypt-before-store: gera **nova chave** (`getNewKey`), encripta a nova senha (`encrypt64`), UPDATE SET UsuKey, UsuSen, **UsuDataRedefinicao**=hoje. Toda troca rotaciona a chave. src `psau_usu_mudasenha.java:71-80,162`.

### CRUD de usuário
- **R12** (validation, high) — UsuNom e UsuLogin obrigatórios. src `sau_usu_impl.java:3247-3260`.
- **R13** (validation, high) — Login **único** (verificado via `psau_verif_usu`: outro UsuCod com mesmo UsuLogin → "Este login já está sendo usado..."). **Não é UNIQUE no banco → impor no serviço.** src `sau_usu_impl.java:3261-3274, psau_verif_usu.java:132`.
- **R14** (validation, high) — FK quando ≠0: UsuProPesCod→SAU_PRO, UsuPrfCod→SAU_PRF, FunPesCod→SAU_FUN. 0 permitido. src `sau_usu_impl.java:3275-3315`.
- **R15** (default, high, **LGPD**) — No INSERT, as **16 permissões de busca de paciente** (UsuPesquisa* Codigo/Data/Nome/DataNasc/CPF/CNS/Prontuario/Endereco + *Operador) default **'0' (negado)**. Gatam por qual identificador o usuário pode buscar pacientes. Devem ser explicitamente concedidas; auditar uso. src `sau_usu_impl.java:10878`.
- **R16** (default, high) — Bloqueio via UsuBloq (checkbox "Bloquear acesso?", 1=bloqueado, enforced no login R5). Tokens default vazios na criação. src `sau_usu_impl.java:450-454,7262-7268`.
- **R17** (validation, high) — Delete bloqueado se referenciado por SAU_USUCON / SAU_USUUNI / (T495) → "CannotDeleteReferencedRecord". src `sau_usu_impl.java:5353-5369`.

### Auditoria
- **R18** (audit, high) — Todo create/update/delete de usuário chama `psau_inc_log` → SAU_LOG (ator=usuário do contexto, LogKey=UsuCod editado). **Mapear para `common/audit` (já implementado na fatia SAU_LOG).** **Sem evento de auditoria de login/logout** encontrado (OQ — LGPD exige auditar autenticação). src `sau_usu_impl.java:5500-5533`.

## Mapeamento JPA (verificado contra o banco vivo — 110 colunas, 1230 linhas, PK usucod, ZERO FK física)

Entidade `Usuario` `@Entity @Table("SAU_USU")` em `…/seguranca/`. **Mapear 16 colunas auth-essenciais** (tabela abaixo); deixar ~94 não mapeadas (validate ignora). `ususen`/`usukey` → `@JsonIgnore`. smallint → `Short @JdbcTypeCode(SqlTypes.SMALLINT)`; `ususysmar` boolean real. Sem `@ManyToOne` (deps não migradas → ids raw). `usulogin` **não é UNIQUE no banco** → unicidade no serviço.

| GX | campo | tipo | anotações |
|---|---|---|---|
| usucod | usuCod | Integer | `@Id @Column(name="UsuCod", nullable=false)` |
| usunom | nome | String | `@Column(name="UsuNom", length=50)` |
| usulogin | login | String | `@Column(name="UsuLogin", length=20)` |
| ususen | senha | String | `@Column(name="UsuSen", length=100) @JsonIgnore` 🔒 |
| usukey | chaveSenha | String | `@Column(name="UsuKey", length=100) @JsonIgnore` 🔒 |
| usutip | tipo | Short | `@Column(name="UsuTip") @JdbcTypeCode(SqlTypes.SMALLINT)` |
| usubloq | bloqueado | Short | `@Column(name="UsuBloq") @JdbcTypeCode(SqlTypes.SMALLINT)` |
| usuprfcod | perfilId | Integer | `@Column(name="UsuPrfCod")` |
| ususysmar | superusuario | Boolean | `@Column(name="UsuSysmar")` |
| usupropescod | profissionalId | Long | `@Column(name="UsuProPesCod")` |
| funpescod | funcionarioId | Long | `@Column(name="FunPesCod")` |
| usutokensoa | tokenSoa | String | `@Column(name="UsuTokenSoa", length=5000)` |
| usutokenexp | tokenExpiracao | Integer | `@Column(name="UsuTokenExp")` |
| usutokendata | tokenData | LocalDateTime | `@Column(name="UsuTokenData")` |
| usudataultimoacesso | ultimoAcesso | LocalDate | `@Column(name="UsuDataUltimoAcesso")` |
| usudataredefinicao | dataRedefinicaoSenha | LocalDate | `@Column(name="UsuDataRedefinicao")` |

## Plano Flyway
- **Produção:** SAU_USU já existe (110 cols) → `validate`, sem ALTER.
- **Baseline (V1):** hoje tem só um **stub de 2 colunas** `SAU_USU(UsuCod, UsuUniCod)` (guard do SAU_UNI) → a entidade **falha o validate**. **Substituir** (V1 linhas ~335-340) pelo subset auth (16 cols + UsuUniCod) abaixo; índice **não-único** `usau_usu` (espelha o vivo — NÃO criar UNIQUE).

```sql
CREATE TABLE IF NOT EXISTS SAU_USU (
    UsuCod INTEGER NOT NULL, UsuNom VARCHAR(50), UsuLogin VARCHAR(20),
    UsuSen VARCHAR(100), UsuKey VARCHAR(100), UsuTip SMALLINT, UsuBloq SMALLINT,
    UsuPrfCod INTEGER, UsuSysmar BOOLEAN, UsuProPesCod BIGINT, FunPesCod BIGINT,
    UsuTokenSoa VARCHAR(5000), UsuTokenExp INTEGER, UsuTokenData TIMESTAMP,
    UsuDataUltimoAcesso DATE, UsuDataRedefinicao DATE, UsuUniCod INTEGER,
    CONSTRAINT pk_sau_usu PRIMARY KEY (UsuCod) );
CREATE INDEX IF NOT EXISTS usau_usu ON SAU_USU (UsuLogin);
```

## Resolved decisions (2026-06-29 — "usar o recomendado")

- **OQ2 ✅ SYSMAR:** mapear o flag **`UsuSysmar=true`** para uma autoridade **`ROLE_SUPERUSER`** (bypassa
  `@PreAuthorize`). **NÃO portar** o caminho mágico `login=='SYSMAR'` (só o flag booleano). Levantar/auditar
  as contas com o flag continua sendo ação de segurança (não bloqueia o migrate).
- **OQ7 ✅ JWT/RBAC:** reusar a infra JWT existente (HS256, `JwtService`, `/auth/login` + `/auth/refresh`).
  Claims: `sub=UsuCod`, `login`, `roles`, `sysmar`, `profissionalId` (escopo PHI futuro). **Mapeamento de
  autoridades COARSE em v1** (a paridade fina por-programa Inc/Alt/Exc/Con fica para uma fatia de autorização
  posterior): no login → `UsuSysmar` ⇒ `ROLE_SUPERUSER`+`SAUDE_ADMIN`+`SAUDE_CADASTRO`; usuário não bloqueado
  com perfil válido ⇒ `SAUDE_CADASTRO` (a role que TODAS as fatias migradas exigem); perfil "admin"
  (via SAU_PRF, OQ8) ⇒ +`SAUDE_ADMIN`. Mantém as fatias migradas funcionando e substitui o login-stub.
- **OQ8 ✅ Introspecção read-only:** autorizado ler **SAU_PRF** (nome do perfil → decidir admin) via query
  nativa, sem migrar suas telas CRUD. **SAU_USUCON/SAU_PRFCON/SAU_USUUNI e o login-por-unidade (R3) ficam
  DEFERIDOS** para a fatia de autorização fina — o login v1 NÃO exige seleção de unidade (simplificação
  documentada vs legado).
- **Perfis de execução:** manter `DevUserDetailsService` (admin/admin123) nos profiles **local/test** (não
  quebrar as ITs/demo); a autenticação real SAU_USU entra no profile **default/prod**. Senha: bcrypt quando
  `usukey IS NULL` (migrado pelo bridge); `usukey` presente ⇒ "redefina a senha" (não migrado).

## 🔴 OPEN QUESTIONS — LISTA DE SIGN-OFF DE SEGURANÇA (resolver ANTES de /migrate-slice)
- **OQ1 ✅ RESOLVIDA (2026-06-28) — bridge via script offline → bcrypt.** Decisão do usuário: converter as
  senhas reversíveis em **bcrypt** num script de migração única, marcando `usukey=NULL` como "migrado" (SEM
  mudança de schema; reusa ususen/usukey). Entregue: **`receituario-modern/tools/password-bridge/`**
  (PasswordBridge.java + pom + README) — decripta via `Encryption.decrypt64(ususen,usukey)` (reflection),
  re-hasheia bcrypt, `UPDATE SET ususen=<bcrypt>, usukey=NULL`. Dry-run default, idempotente, auto-verifica,
  nunca loga plaintext. **Bloqueio (→ OQ3):** precisa do **jar do runtime GeneXus** (`com.genexus.util.Encryption`)
  no classpath — NÃO está no repo (só no servidor legado); rodar no host legado OU fornecer o jar.
  AuthService: usukey NULL→bcrypt; usukey presente→"redefina a senha" (não migrado).
- **OQ2 ⚠ (backdoors SYSMAR):** dois caminhos incondicionais (UsuSysmar boolean + login=='SYSMAR'). Levantar quem os possui; decidir manter/gate/remover antes do cutover.
- **OQ3 ✅ RESOLVIDA (2026-06-29):** o runtime do GeneXus está em
  **`Receituario/JavaModel/web/gxclassR.zip`** (classes Runtime; era `.zip`, por isso a busca por `.jar`
  não achou). Contém `com.genexus.util.Encryption.decrypt64(String,String)` — um zip serve no classpath.
  Bridge rodado com `-Dgenexus.jar.path=…/gxclassR.zip`. **DRY-RUN COMPLETO validado: 1228/1228 decriptadas
  + bcryptadas + auto-verificadas, 0 falhas, nada escrito.** Approach 100% comprovado contra o snapshot.
  ⚠ NÃO rodar `--commit` ainda: (1) só fazer no CUTOVER de auth (depois do /migrate-slice SAU_USU), senão
  ninguém loga via SAU_USU; (2) quebra o login do app LEGADO contra este snapshot (ususen vira bcrypt,
  usukey NULL → decrypt64 legado falha) — exige coordenação + backup do SAU_USU.
- **OQ4 (lockout/expiração):** sem contador de tentativas/lockout/expiração inline no login (apesar de ParInaUsuDias/ParSenUsuDias carregados). Confirmar se expiração/inatividade (em halertadataexpiracao/desbloqueiausuario) deve ser portada.
- **OQ5 (auditoria de login/logout):** psau_inc_log dispara no CRUD de usuário, mas **nenhum evento de login/logout** foi encontrado. LGPD exige auditar autenticação — fechar essa lacuna no app moderno (usar common/audit).
- **OQ6 (UsuTokenSoa):** coluna de 5000 chars — token SOA/SSO? Confirmar se algum sistema externo a lê (manter) ou se é morta (ignorar).
- **OQ7 (JWT + RBAC mapping):** claims (sub=UsuCod, login, perfil/roles, sysmar, profissionalId p/ escopo PHI), TTL access/refresh, chave por env. Projeção de (UsuPrfCod→SAU_PRF + SAU_PRFCON/SAU_USUCON Inc/Alt/Exc/Con + bitmap UsuBloq*) → GrantedAuthorities/@PreAuthorize. Confirmar polaridade de UsuBloq*.
- **OQ8 (deps não migradas):** RBAC real precisa ler SAU_PRF/USUCON/PRFCON/PRG e SAU_USUUNI (login por unidade, R3). Decidir: introspecção read-only dessas tabelas para o login, sem migrar suas telas CRUD ainda. Confirmar se o login moderno exige seleção de unidade (R3) ou se o escopo de unidade muda no modelo JWT.
- **OQ9 (LGPD usupesquisa*):** 16 flags de permissão de busca de paciente (default negado). Definir enforcement server-side em /api/pacientes e auditar concessões/mudanças.
- **OQ10 (login uniqueness):** não há UNIQUE no banco (só índice não-único) → impor no serviço (findByLogin ≤1).
```
