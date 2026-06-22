<!-- refreshed: 2026-05-21 -->
# Architecture

**Analysis Date:** 2026-05-21

## System Overview

```text
┌───────────────────────────────────────────────────────────────────────────┐
│                        GeneXus KB (source of truth)                       │
│                                                                           │
│  Transactions | Procedures | Data Providers | Web Panels | SDTs | BCs     │
│  (lives in GeneXus IDE — NOT in this repo; this repo is generator output) │
└────────────────────────────────┬──────────────────────────────────────────┘
                                 │ GeneXus Java Generator 15_0_7-118219
                                 │ (`GXJMake.exe`, `GXSPRO12.002`,
                                 │  `GXTPRO12.002`, `GXPPRO12.002`)
                                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│       Generated Java sources — `Receituario/JavaModel/web/*.java`         │
├──────────────┬─────────────┬──────────────┬─────────────┬─────────────────┤
│ Transactions │ Web Panels  │ Procedures   │ SDTs / BCs  │ Master Pages    │
│ `sau_*.java` │ `hww*.java` │ `psau_*.java`│ `Sdt*.java` │ `appmasterpage` │
│ `sau_*_impl` │ `hview*`    │ `r*.java`    │ `*_bc.java` │ `hmaster*.java` │
│              │ `hprompt*`  │ (HTTP procs) │             │ `rwdmaster*`    │
└──────┬───────┴──────┬──────┴───────┬──────┴──────┬──────┴────────┬────────┘
       │              │              │             │               │
       └──────────────┴──────────────┴─────────────┴───────────────┘
                                 │ javac (companion `.class` in
                                 │  `WEB-INF/classes/`, see Tomcat copy)
                                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│              Tomcat 8.5 deployment — `ReceituarioJavaEnvironment/`         │
│                                                                           │
│  `WEB-INF/web.xml` (servlet wiring + GeneXus runtime listeners)            │
│  `WEB-INF/classes/*.class` (665 compiled objects)                          │
│  `WEB-INF/lib/*.jar` (61 jars incl. `GeneXus.jar`, `gxclassR.jar`)         │
│  `static/`, `gxmetadata/`, `Resources/` (assets, metadata, theme)          │
└────────────────────────────────┬──────────────────────────────────────────┘
                                 │ HTTP request `/servlet/<object>`
                                 │ handled by `@WebServlet`-annotated stub
                                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│   GeneXus runtime: `com.genexus.webpanels.*` + `GXDataArea` per object     │
│   (`GxWebStd.java`, `SdtContext.java`, `GxObjectCollection.java`)          │
└────────────────────────────────┬──────────────────────────────────────────┘
                                 │ JDBC via cursors declared in
                                 │  `<object>__default.class`
                                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│   PostgreSQL database `saude-mandaguari` @ 127.0.0.1:5432                  │
│   (driver: `org.postgresql.Driver`; see `client.cfg` line 113-114)         │
└───────────────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| Transaction stub (servlet entry) | Map `/servlet/<name>` URL to runtime; declare integrated-security flags | `sau_pac.java`, `sau_rem.java`, `sys_pes.java` |
| Transaction impl (`*_impl.java`) | Form lifecycle: load/save, Ajax handlers, business rules generated from KB | `sau_pac_impl.java` (947 KB), `sau_pesf_pac_impl.java` |
| Business Component (`*_bc.java`) | Headless callable trn — programmatic CRUD for the same transaction | `sau_pac_bc.java`, `sau_log_bc.java`, `sau_remlot_bc.java`, `sys_pes_bc.java` |
| Work-With panel (`hww*.java`) | Grid/list UI: filter, paginate, navigate to view/edit | `hwwsau_pac.java`, `hwwsau_rem.java`, `hwwsau_fun.java` |
| View panel (`hview*.java`) | Read-only detail of one record | `hviewsau_pac.java`, `hviewsau_rem.java` |
| Prompt panel (`hpromptsau_*.java`) | Modal lookup for selecting an FK value | `hpromptsau_pac.java`, `hpromptsau_cep.java` (58 files) |
| Section panel (`hsau_*general.java`, `*wc.java`) | Tab/section embedded in a transaction form or work-with | `hsau_pacgeneral.java`, `hsau_prggrpsau_prgwc.java` |
| Procedure (`p*.java`) | Batch / callable server logic, no UI; mostly DB queries | `psau_inc_dis.java`, `psau_busca_cod.java` (56 `psau_*` + 14 other `p*`) |
| HTTP procedure / report (`r*.java`) | URL-callable procedure, often emits PDF (uses `iText.jar`) | `rsau_recesp.java`, `rsau_pacextdet.java` |
| Data accessor (`*__default.class`) | All SQL cursor declarations for one object; loaded by impl class | `sau_pac__default.class`, `psau_inc_dis__default.class` |
| SDT (`Sdt*.java`) | Structured Data Type — typed value object passed between objects | `SdtSAU_PAC.java`, `SdtContext.java`, `SdtSDT_INFO_PACIENTE_Item.java` |
| Domain enum (`gxdomain*.java`) | Enumerated domain values from the KB | `gxdomaintrnmode.java`, `gxdomainnotperfil.java` |
| Master page (`appmasterpage`, `rwdmasterpage`, `hmaster*`) | Layout shell wrapping content panels | `appmasterpage.java`, `rwdmasterpage.java`, `hmasterpagelogin.java` |
| Runtime stdlib (root `Gx*.java`) | Shared helpers (collections, web std, full-text reindex) | `GxWebStd.java` (238 KB), `GxObjectCollection.java`, `GxFullTextSearchReindexer.java` |
| Config (`GXcfg.java`) | Wired into `web.xml` as `<context-param>gxcfg</context-param>` | `GXcfg.java`, `client.cfg` |

## Pattern Overview

**Overall:** Model-driven code generation — GeneXus KB (Knowledge Base) is the source of truth; every `.java` file in this tree is regenerated by the GeneXus Java Generator and should never be hand-edited.

**Key Characteristics:**
- Two-class-per-object split: a thin `<name>.java` servlet stub extending `GXWebObjectStub` + a fat `<name>_impl.java` extending `GXDataArea` (transactions/web panels) or `GXProcedure` (procedures). Sample stub: `sau_pac.java` (48 lines); sample impl: `sau_pac_impl.java` (947 KB).
- A third companion class `<name>__default.class` carries the JDBC cursor declarations used by that object (visible in `WEB-INF/classes/`).
- Headers in every generated file:
  ```
  File: SAU_PAC
  Description: Paciente
  Author: GeneXus Java Generator version 15_0_7-118219
  Generated on: May 20, 2026 ...
  Program type: Callable routine
  Main DBMS: PostgreSQL
  ```
- Servlet wiring via `@javax.servlet.annotation.WebServlet(value="/servlet/<name>")` annotations on every public object → no per-object `<servlet>` entries in `web.xml`.
- All inter-object calls go through the GeneXus runtime (`com.genexus.webpanels.*`, `com.genexus.db.*`); the generated code rarely uses standard Java idioms (no Spring, no JPA, no Jackson on domain objects).
- Ajax interactions are handled by Action numbers embedded in `_impl` switch ladders, e.g. `"gxJX_Action75"` in `sau_pac_impl.java:43`.

## Layers

**Layer 1 — GeneXus KB (external, source of truth):**
- Purpose: Define transactions, procedures, web panels, SDTs declaratively.
- Location: NOT in this repo. Lives in the GeneXus IDE elsewhere on the developer machine.
- Used by: GeneXus Java Generator at "specify / generate" time.

**Layer 2 — Generator binaries:**
- Purpose: Translate KB objects into Java + JS + CSS.
- Location: Root of `/mnt/s/projetos/Receituario/JavaModel/web/`.
- Files: `GXJMake.exe`, `GXSPRO12.002`, `GXTPRO12.002`, `GXPPRO12.002`, `GXHPRO12.002`, `GX_PRO12.002`, `CALLMAKE.BAT`, `GXScanner.jar`.

**Layer 3 — Generated Java sources:**
- Purpose: Compilable output mirroring each KB object.
- Location: `/mnt/s/projetos/Receituario/JavaModel/web/*.java` (468 .java files in root).
- Contains: Servlet stubs, `_impl` bodies, BCs, SDTs, domains, master pages.
- Depends on: `com.genexus.*` runtime jars (Layer 5).
- Used by: javac → `.class` → Tomcat (Layer 6).

**Layer 4 — Object metadata:**
- Purpose: Describe table dependencies per object and object schema per BC.
- Location: `Metadata/TableAccess/*.xml` (142 files; one per object — example `Metadata/TableAccess/AcessaModulo.xml` lists `SAU_USU`, `SAU_USUUNI`), and `gxmetadata/*.json` (per-BC schema — `gxmetadata/sau_pac.json`, `gxmetadata/sau_log.json`, `gxmetadata/sau_remlot.json`, `gxmetadata/sys_pes.json`).
- Used by: GeneXus tooling, reorg, runtime introspection.

**Layer 5 — GeneXus runtime libraries:**
- Purpose: Hosting classes that every generated object extends or calls.
- Location: `modules/GeneXus.jar`, `services/GXWebSocket.jar`, deployed `WEB-INF/lib/*.jar` (61 jars including `gxclassR.jar`, `iText.jar`, `bcprov-jdk15on-147.jar`, `commons-fileupload-1.3.2.jar`).
- Used by: every `*.java` file (`import com.genexus.*;` is the first import in all of them).

**Layer 6 — Tomcat deployment:**
- Purpose: Serve the compiled webapp.
- Location: `/mnt/c/Program Files/Apache Software Foundation/Tomcat 8.5/webapps/ReceituarioJavaEnvironment/`.
- Contains: `WEB-INF/web.xml`, `WEB-INF/classes/*.class` (665 files), `WEB-INF/lib/*.jar`, `static/`, `gxmetadata/`, `Metadata/`, `LayoutMetadata/`, `PublicTempStorage/`.

**Layer 7 — PostgreSQL database:**
- Purpose: System of record.
- Connection: `jdbc:postgresql://127.0.0.1:5432/saude-mandaguari` (`client.cfg:114`).
- Driver: `drivers/postgresql-9.1-902.jdbc3.jar`.
- NOTE: The prompt mentioned "SQL Server `sysmar`" but `client.cfg` explicitly sets `DBMS=postgresql` and `JDBC_DRIVER=org.postgresql.Driver`. The "Sysmar" name surfaces only as the UI theme `Theme=SysmarTheme` (`client.cfg:59`).

## Data Flow

### Primary Request Path (HTML form roundtrip)

1. Browser issues `GET/POST /ReceituarioJavaEnvironment/servlet/sau_pac?<gxfirstwebparm>` — Tomcat routes by `@WebServlet` annotation to `sau_pac.java:15`.
2. `sau_pac.doExecute()` (`sau_pac.java:18-21`) constructs `new sau_pac_impl(context).doExecute()` — the stub forwards immediately to the impl.
3. `sau_pac_impl.inittrn()` (`sau_pac_impl.java:25`) reads `gxfirstwebparm` and dispatches:
   - `"dyncall"` → Ajax dynamic call (`sau_pac_impl.java:32-42`).
   - `"gxJX_Action<N>"` → one of dozens of action handlers (`sau_pac_impl.java:43-100+`).
   - Default → render form (load via cursors declared in `sau_pac__default.class`).
4. SQL is executed through `pr_default.execute(<cursorIndex>)` (e.g. `psau_inc_dis.java:55`) — cursors are pre-declared in the `__default` helper.
5. Response is built by `httpContext` (a `com.genexus.internet.HttpContext`) which writes HTML; client-side JS (`sau_pac.js`, `appmasterpage.js`) handles widgets.

### Procedure call path (no UI)

1. URL `/servlet/rsau_recesp?<params>` → `rsau_recesp.java:15` (an HTTP procedure).
2. `rsau_recesp_impl.execute()` runs DB queries, builds a PDF via `iText.jar`, writes it to the response stream.
3. Internal callers reach procedures with `new psau_inc_dis(remoteHandle).execute(byte[] aP0)` — `byte[]`/scalar inout params, no exceptions thrown across boundaries.

### Master-page composition

1. A transaction servlet (e.g. `sau_pac`) is bound to a master page declared in the KB.
2. Master page implementations: `appmasterpage_impl.java`, `rwdmasterpage_impl.java`, `hmasterpagetransaction_impl.java`, `hmasterpageww_impl.java`, `hmasterpagelogin_impl.java`, `hmasterpageprompt_impl.java`, `hmasterpagesuporte_impl.java`.
3. At render time the impl includes header/footer/menu fragments (`hpageheader.java`, `hpagefooter.java`, `hmenusuporte.java`).

**State Management:**
- HTTP session via `com.genexus.webpanels.SessionEventListener` (`WEB-INF/web.xml:15-17`).
- Per-request context via `SdtContext` (`SdtContext.java`, 144 KB — the catch-all environment SDT).
- "Recent links" memory via `recentlinks.java`, `rwdrecentlinks.java`, `SdtLinkList_LinkItem.java`.
- No JWT, no Spring Security. Authentication is handled inside generated objects (`acessamodulo.java`, `cadastrarsenha.java`, `bloqueiausuario.java`, `desbloqueiausuario.java`, `hmudasenhalogin.java`).

## Key Abstractions

**Transaction (PRIMARY ABSTRACTION):**
- A business object representing one or more related tables plus its rules, formulas, and UI form. This is the core architectural unit in GeneXus — everything else exists to serve transactions.
- Examples in this codebase (35 transactions discovered, all under the `sau_*` and `sys_*` domain prefixes):
  - `sau_pac.java` + `sau_pac_impl.java` — Paciente (Patient)
  - `sau_rem.java` + `sau_rem_impl.java` — Remédio (Medicine)
  - `sau_recesp.java` + `sau_recesp_impl.java` — Receita Especial (Prescription)
  - `sau_pesf.java` + `sau_pesf_impl.java` — Profissional de Saúde
  - `sau_pesf_pac.java` — nested transaction (profissional ↔ paciente vinculo)
  - `sys_pes.java`, `sys_emp.java` — system-level person/employer
- Pattern: each transaction has *up to* this constellation of objects:
  - `<trn>.java` + `<trn>_impl.java` (form servlet)
  - `<trn>_bc.java` (headless business component, only for some — 4 detected: `sau_log_bc`, `sau_pac_bc`, `sau_remlot_bc`, `sys_pes_bc`)
  - `hww<trn>.java` (Work-With grid)
  - `hview<trn>.java` (read-only View)
  - `hprompt<trn>.java` (FK lookup)
  - `hsau_<trn>general.java` (general tab section)
  - `Sdt<TRN_NAME>.java` (structure when used as a SDT)
  - `Metadata/TableAccess/<Trn>.xml` (table dependencies)
  - `gxmetadata/<trn>.json` (BC schema, only for BC-enabled transactions)

**Procedure:**
- Server-side callable subroutine. Two flavors:
  - **Internal procedure** (program type `Callable routine`) — invoked by other GX objects only. Extends `GXProcedure`. Example: `psau_inc_dis.java:14` (`extends GXProcedure`).
  - **HTTP procedure** (program type `HTTP procedure`) — URL-addressable, `@WebServlet`-annotated. Extends `GXWebObjectStub`. Example: `rsau_recesp.java:16`.
- Naming: `psau_*` (most), plus action-named `acessamodulo`, `bloqueiausuario`, `buscaibge`, `pacientecarregainfo`, etc.

**Web Panel:**
- Stateless UI screen (not directly tied to a single table). Same stub/impl split. Examples: `hindex.java`, `hliberacao.java`, `home.java`, `popupatencao_interacaomedicamentosa.java`, `cadastrasenha1.java`.

**Data Provider (DP):**
- Not detected in this generated output. The GeneXus naming `dp*.java` produced zero matches; data lookups are done either inline in `_impl` files or by dedicated procedures (`busca*`, `existe*`, `pfindtab.java`).

**SDT (Structured Data Type):**
- Typed record/collection passed between GX objects. 19 detected:
  - `SdtContext.java` — request/session context (144 KB).
  - `SdtSAU_PAC.java`, `SdtSAU_LOG.java`, `SdtSAU_REMLOT.java`, `SdtSYS_PES.java` — BC-mirrored SDTs.
  - `SdtSDT_INFO_PACIENTE_Item.java`, `SdtSDT_QTD_Item.java`, `SdtSDT_SAU_PESF_PAC.java`, `SdtSDT_SAU_REMLOT_Item.java`, `SdtSDT_TipoReceita_SDT_TipoReceitaItem.java` — custom DTOs.
  - `SdtSeguranca.java`, `SdtSlideDownMenuData_*`, `SdtTabOptions_*`, `SdtTransactionContext*` — UI/security plumbing.
  - `SdtLinkList_LinkItem.java`, `SdtProgramNames_ProgramName.java` — runtime indexes.

**Business Component (BC):**
- A transaction exposed as a programmable object (Save/Load/Delete via setters). Only 4 transactions have BCs: `sau_pac_bc.java:15` (`extends GXWebPanel implements IGxSilentTrn`), `sau_log_bc`, `sau_remlot_bc`, `sys_pes_bc`. Their schemas are persisted in `gxmetadata/*.json`.

**Domain (`gxdomain*`):**
- Enumerated value sets defined in the KB and emitted as classes: `gxdomaintrnmode.java`, `gxdomainboolean90.java`, `gxdomaincores.java`, `gxdomainsus_rendafamemsalsmins.java`, `gxdomaintiposdecotas.java`, `gxdomaintiposdeprocs.java`, etc.

## Entry Points

**Servlet listeners (declared in `WEB-INF/web.xml:11-17`):**
- `com.genexus.webpanels.ServletEventListener` — boots the GeneXus runtime on context startup.
- `com.genexus.webpanels.SessionEventListener` — wires HTTP session lifecycle.

**Infrastructure servlets (`WEB-INF/web.xml:19-122`):**
| URL | Servlet class | Purpose |
|-----|---------------|---------|
| `/gxobject` | `com.genexus.webpanels.GXObjectUploadServices` | File uploads from web panels |
| `/oauth/access_token` | `GXOAuthAccessTokenDummy` | OAuth (disabled — Dummy stubs) |
| `/oauth/logout` | `GXOAuthLogoutDummy` | OAuth logout (disabled) |
| `/oauth/userinfo` | `GXOAuthUserInfoDummy` | OAuth userinfo (disabled) |
| `/gx_valid_service` | `GXValidService` | Client-side validation callback |
| `/servlet/com.genexus.webpanels.GXResourceProvider` | `GXResourceProvider` | Static resource server |
| `/oauth/gam/*` | `agamextauthinputDummy`, `agamoauth20get*Dummy` | GeneXus Access Manager (disabled) |
| `/servlet/com.genexus.webpanels.gxver` | `com.genexus.webpanels.gxver` | Version probe |

**Business servlets (NOT in `web.xml` — registered via `@WebServlet` annotation):**
- Every generated `<name>.java` that extends `GXWebObjectStub` registers itself at `/servlet/<name>`. Examples: `sau_pac.java:15` → `/servlet/sau_pac`; `home.java:15` → `/servlet/home`; `hwwsau_pac.java:15` → `/servlet/hwwsau_pac`; `rsau_recesp.java:15` → `/servlet/rsau_recesp`.
- Default landing page (KB Start Object): `home.java` (`/servlet/home`).
- Login flow: `hmudasenhalogin.java`, `cadastrarsenha.java`, `cadastrasenha1.java`, `acessamodulo.java`, `bloqueiausuario.java`.

**Filters (`WEB-INF/web.xml:129-149`):**
- `com.genexus.filters.ExpiresFilter` — sets cache headers (`access plus 36 hours`) on `image`, `text/css`, `application/javascript`.

**Context params:**
- `gxcfg` → `GXcfg` (`web.xml:124-127`) — bootstraps runtime to read `client.cfg`.

**REST endpoints:**
- None detected. No classes named `*REST*`, `*Rest*`, or `GXRESTSPC` exist in either the source tree or the deployed `WEB-INF/classes`. This webapp is HTML-form / servlet only.

## Architectural Constraints

- **Generated, not hand-written:** Every `*.java` in the root is regenerated by the GeneXus Java Generator (header banner asserts this). Hand-editing is destroyed on the next `Build/Run` from the GeneXus IDE. The only way to change behavior is to edit the KB and regenerate.
- **No package layout:** Every class is in the *default package* — there are no `package` declarations in any generated file (verified in `home.java`, `sau_pac.java`, `psau_inc_dis.java`, `GXcfg.java`). `WEB-INF/classes/` is a flat directory of 665 `.class` files. New code added to the GeneXus model will continue this convention; manual classes cannot use a package without breaking the generator's classloading.
- **Threading:** Standard Tomcat thread-per-request. `SessionEventListener` keeps per-session state in the servlet container; per-request state lives in `HttpContext`.
- **Global state:** `ModelContext` (per-classloader), `client.cfg` (process-wide), `GXcfg` (context-param wired in `web.xml:124-127`).
- **Database is PostgreSQL** (`client.cfg:113-114`, `121`). Every generated file header says `Main DBMS: PostgreSQL`. Switching DBMS requires re-running the GeneXus reorg + regenerate.
- **Connection pooling:** `PoolRWEnabled=1`, `POOLSIZE_RW=10`, `UnlimitedRWPool=1`, `RecycleRW=1` (`client.cfg:122-127`). Submit pool size = 5 (`client.cfg:61`).
- **Encryption:** `USE_ENCRYPTION=SESSION` (`client.cfg:62`) — request parameters are encrypted/signed per session via `httpContext.DecryptAjaxCall(...)` (visible in `sau_pac_impl.java:31`).
- **Localization:** Hard-wired to Brazilian Portuguese (`LANGUAGE=por`, `culture=pt-BR`, `DATE_FMT=DMY`, `DECIMAL_POINT=,` in `client.cfg:57-58, 82-88`). UI strings live under `Resources/Portuguese/`.
- **Theme:** `Theme=SysmarTheme` (`client.cfg:59`) — assets under `Resources/SysmarTheme/` and `Resources/Portuguese/SysmarTheme/`.

## Anti-Patterns

### Editing `*.java` files directly

**What happens:** A developer modifies `sau_pac_impl.java` or any other generated file to fix a bug or add behavior.
**Why it's wrong:** The next regen from the GeneXus IDE overwrites the file; banner `Generated on:` makes this explicit. All 468 root `.java` files are owned by the generator.
**Do this instead:** Locate the corresponding KB object in the GeneXus IDE (transaction `SAU_PAC`, procedure `PSAU_INC_DIS`, etc.), modify its rules/events/form, then "Build/Run" to regenerate.

### Adding new packages or `package` statements

**What happens:** A developer creates a custom `.java` with `package com.example.helper;` and drops it under `/web/`.
**Why it's wrong:** The generator emits flat default-package classes (verified in `home.java`, `sau_pac.java`) and Tomcat's `WEB-INF/classes/` is a flat tree. The generator does not scan subpackages for KB-referenced names; cross-calls will fail.
**Do this instead:** Add custom logic as a KB Procedure or an External Object pointing to a custom jar dropped into `WEB-INF/lib/`.

### Treating `Metadata/`, `gxmetadata/`, and `LayoutMetadata/` as editable

**What happens:** Manual edit of `Metadata/TableAccess/*.xml` or `gxmetadata/*.json` to "fix" a missing dependency.
**Why it's wrong:** These are generator outputs (XML has GUIDs that get regenerated; JSON mirrors KB structure). Example `Metadata/TableAccess/AcessaModulo.xml` is recomputed from object analysis.
**Do this instead:** Adjust the KB object's referenced attributes/tables; the metadata gets re-emitted.

### Storing secrets in `client.cfg`

**What happens:** `USER_ID=qVxDieU5ngJ+7cCSaUXFYk==` and `USER_PASSWORD=dudhggSQtliZfx4F86JMacC8tMkXXXE9WGMHMnI+K5w=` (`client.cfg:107-108`) hold the DB credentials, encoded but committed.
**Why it's wrong:** The encoded values are reversible by the GeneXus runtime (it must decode them at startup); they are effectively in plaintext for anyone with the runtime jars. The file is also duplicated into the Tomcat deployment.
**Do this instead:** Switch the KB to use a JDBC DataSource (`USE_JDBC_DATASOURCE=1`, `JDBC_DATASOURCE=jndi/...`) configured in Tomcat's `server.xml` so secrets live outside the webapp.

### Calling cross-object methods directly instead of through the runtime

**What happens:** Custom code instantiates `new sau_pac_impl(context)` and calls private-looking methods.
**Why it's wrong:** Object lifecycles (init, executeUdp, privateExecute, cleanup) are runtime-managed; bypassing them skips cursor open/close, transaction scope, security checks (`IntegratedSecurityEnabled` on every stub).
**Do this instead:** Call procedures via their constructor + `execute(byte[])` (see `psau_inc_dis.java:36-39`), or BCs via their public CRUD methods (see `sau_pac_bc.java:33-50`).

## Error Handling

**Strategy:** Single byte-flag pattern — `GxWebError` is set on validation/Ajax failures and the impl returns early. Example: `sau_pac_impl.java:36-38`:
```
if ( ! httpContext.IsValidAjaxCall( true) )
{
   GxWebError = (byte)(1) ;
   return  ;
}
```

**Patterns:**
- No checked exceptions thrown across object boundaries — procedure `execute` returns `void` with inout `byte[]` for status (see `psau_inc_dis.java:36-38`).
- Form-level errors collected through GeneXus's `Msg` API and rendered into the master page.
- "Not authorized" pages: `hnotauthorized*.java` (7 variants for different masterpages: `_impl`, `prompt_impl`, `system_impl`, `user_impl`, `suporte_impl`, `promptuser_impl`).
- Servlet-level exceptions surface as 500s via Tomcat's default error pipeline (no `<error-page>` entries in `web.xml`).

## Cross-Cutting Concerns

**Logging:**
- JDBC logging gated by `client.cfg:93-99` (`JDBCLogEnabled=0`, `JDBCLogLevel=0`). Off by default.
- Application-level audit is a domain table — see `sau_log.java` + `sau_log_bc.java` + `gxmetadata/sau_log.json` (audit business component).
- Runtime logs to Tomcat's `catalina.out` via the GeneXus runtime's own logger.

**Validation:**
- Client-side: `GXValidService` servlet (`web.xml:37-40`, mapping `/gx_valid_service`) — invoked by JS for live field validation.
- Server-side: emitted inline in `*_impl.java` (`gxJX_ActionN` ladders run the trn rules and re-assign attributes).
- Domain-typed values use `gxdomain*.java` classes.

**Authentication:**
- Integrated Security is **disabled**: `EnableIntegratedSecurity=0` (`client.cfg:78`); every generated stub returns `IntegratedSecurityEnabled() = false` (see `sau_pac.java:33-36`, `home.java:33-36`, `hwwsau_pac.java:33-36`).
- Custom auth instead: `acessamodulo.java` (module access), `cadastrarsenha.java` + `cadastrasenha1.java` (password setup), `hmudasenhalogin.java` (change password), `bloqueiausuario.java` / `desbloqueiausuario.java`, `pisauthorized.java`, `psau_checarperm.java`. Backing tables: `SAU_USU`, `SAU_USUUNI` (see `Metadata/TableAccess/AcessaModulo.xml`).
- OAuth/GAM servlets in `web.xml` are all `*Dummy` classes — placeholders, not wired to a real provider.

**Caching:**
- `ExpiresFilter` (`web.xml:129-149`) — 36-hour cache for images, CSS, JS.
- Application cache disabled: `CACHING=0`, `SMART_CACHING=0` (`client.cfg:68-69`).
- Cache invalidation token: `CACHE_INVALIDATION_TOKEN=20265201014253` (`client.cfg:70`).

**Internationalization:**
- Single language (Portuguese-BR). String tables under `Resources/Portuguese/`. Locale config in `client.cfg:82-88`.

**WebSocket / push:**
- `services/GXWebSocket.jar` enabled via `context.xml:4` (`pluggabilityScan="GXWebSocket.jar"`).
- `CloudServices.config` wires `WebNotifications` to `com.genexus.internet.websocket.GXWebSocket`. Handlers are blank → no app-level notification handler is registered.

**File uploads / temp storage:**
- `PublicTempStorage/` (BLOB output, `client.cfg:60`).
- `TMPMEDIA_DIR=PrivateTempStorage` (`client.cfg:51`).
- Layout metadata for PDF/printable templates: `LayoutMetadata/` (in deployed copy, `client.cfg:52`).

---

*Architecture analysis: 2026-05-21*
