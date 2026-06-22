# Codebase Structure

**Analysis Date:** 2026-05-21

## Directory Layout

```text
/mnt/s/projetos/Receituario/JavaModel/web/        # GeneXus Java regen output (1446 files)
├── *.java                                         # Generated objects (468 files, flat default package)
├── *.class                                        # Local compile artifacts (companion to .java)
├── *.js                                           # Client-side companions per object (e.g. sau_pac.js)
├── *.css                                          # Theme/component stylesheets
├── *.002 / *.DLL / *.exe / *.VER / *.BAT         # GeneXus generator binaries (Windows)
├── client.cfg                                     # Runtime config — DB URL, locale, theme, pools
├── context.xml                                    # Tomcat per-context overrides (JarScanner)
├── context*.xml                                   # JarScanner variants
├── web.xml                                        # Servlet/listener/filter wiring
├── web6*.xml                                      # Variants for older Servlet specs
├── developermenu.html                             # GeneXus dev menu (devmenu/ assets)
├── gx_blank.html                                  # Empty page stub
├── CloudServices.config                           # WebNotifications/WebSocket service binding
├── GeneXus.services                               # Exposable runtime services manifest
├── Images.txt                                     # Image manifest
├── LastCallTree.info                              # Last-build call graph dump
├── Metadata/                                      # Per-object table-access XML (142 files)
│   └── TableAccess/                               # <ObjectName>.xml → list of tables touched
├── gxmetadata/                                    # Per-BC JSON schemas (4 BCs + gxversion.json)
│   ├── gxversion.json                             # GeneXus runtime version
│   ├── sau_log.json                               # SAU_LOG business component schema
│   ├── sau_pac.json                               # SAU_PAC business component schema
│   ├── sau_remlot.json                            # SAU_REMLOT business component schema
│   └── sys_pes.json                               # SYS_PES business component schema
├── Resources/                                     # Theme + i18n assets
│   ├── GeneXusXEv2/                               # Default GX theme
│   ├── Portuguese/                                # i18n (pt-BR) string + image assets
│   │   ├── GeneXusXEv2/
│   │   ├── SimpleAndroid/
│   │   └── SysmarTheme/
│   ├── SimpleAndroid/                             # Mobile-ish theme
│   └── SysmarTheme/                               # Active theme (client.cfg:59)
├── modules/                                       # GeneXus runtime jar staging
│   └── GeneXus.jar
├── services/                                      # Service jars
│   └── GXWebSocket.jar
├── drivers/                                       # JDBC driver jars (all DBs supported by GX)
│   ├── jt400.jar                                  # AS/400
│   ├── jtds-1.2.jar                               # SQL Server (NOT used at runtime)
│   ├── mysql-connector-java-5.1.11-bin.jar
│   ├── ojdbc6.jar                                 # Oracle
│   ├── postgresql-9.1-902.jdbc3.jar               # *** Active driver (client.cfg:113) ***
│   └── sqlitejdbc-v056.jar
├── bootstrap/                                     # Bootstrap CSS/JS framework (static)
├── devmenu/                                       # GeneXus developer menu UI (icons, JS, CSS)
├── HMask/                                         # Field masking widget (jQuery + meiomask)
├── SlideDownMenu/                                 # Slide-down menu component (sdmenu)
└── GxMask-master/                                 # Mask plugin (vendor copy)
```

```text
/mnt/c/Program Files/Apache Software Foundation/Tomcat 8.5/webapps/ReceituarioJavaEnvironment/
├── WEB-INF/
│   ├── web.xml                                    # Same as JavaModel/web/web.xml
│   ├── classes/                                   # 665 compiled .class files (flat, no packages)
│   │   ├── <object>.class                         # Compiled stub (e.g. sau_pac.class)
│   │   ├── <object>_impl.class                    # Compiled impl
│   │   ├── <object>__default.class                # SQL cursor declarations for the object
│   │   └── client.cfg                             # Runtime-loaded copy of client.cfg
│   ├── lib/                                       # 61 third-party jars
│   │   ├── GeneXus.jar                            # GeneXus runtime
│   │   ├── gxclassR.jar                           # Reports runtime
│   │   ├── GXWebSocket.jar                        # WebSocket transport
│   │   ├── iText.jar / iTextAsian.jar             # PDF generation
│   │   ├── bcprov-jdk15on-147.jar                 # BouncyCastle crypto
│   │   ├── commons-fileupload-1.3.2.jar           # File uploads
│   │   ├── jackson-annotations-2.7.3.jar          # JSON (selective use)
│   │   └── ...                                    # commons-*, dom4j, hk2, asm, activation, ...
│   ├── CloudServices.config
│   ├── GXPRN.INI                                  # Print/report config
│   └── PDFReport.ini                              # PDF report config
├── gxmetadata/                                    # Mirror of source gxmetadata/
├── Metadata/                                      # Mirror of source Metadata/
├── LayoutMetadata/                                # PDF/print layout templates (deploy-only)
├── META-INF/                                      # Standard webapp META-INF
├── PublicTempStorage/                             # Public BLOB output (client.cfg:60)
├── static/                                        # Static assets exposed at /static/* (client.cfg:46)
│   ├── bootstrap/
│   ├── devmenu/
│   ├── GxMask-master/
│   ├── HMask/
│   ├── Metadata/                                  # Public copy of metadata
│   ├── Resources/                                 # Public copy of theme
│   └── SlideDownMenu/
├── GXDIB32.DLL                                    # Windows native imaging
├── printingappletsigned.jar                       # Legacy signed applet for client-side print
└── rbuildj.dll                                    # Windows native helper
```

## Directory Purposes

**`/mnt/s/projetos/Receituario/JavaModel/web/` (source root):**
- Purpose: Output of the GeneXus Java Generator. NOT hand-written. Every `.java` carries a banner identifying the generator.
- Contains: Generated `.java`/`.class`/`.js`/`.css` per object; generator binaries; runtime config; metadata.
- Key files: `client.cfg`, `web.xml`, `GXcfg.java`, every `sau_*.java`.

**`/mnt/s/projetos/Receituario/JavaModel/web/Metadata/TableAccess/`:**
- Purpose: One XML file per object listing the DB tables it depends on (for reorg, impact analysis).
- Contains: 142 `*.xml` files; example `AcessaModulo.xml` declares dependencies on `SAU_USU`, `SAU_USUUNI`.
- Generated: Yes. Committed: Yes (part of regen output).

**`/mnt/s/projetos/Receituario/JavaModel/web/gxmetadata/`:**
- Purpose: JSON schema for each Business Component (BC) — used by the GX runtime to expose BCs as JSON.
- Contains: `gxversion.json` plus one `<bc>.json` per BC (4 detected: `sau_log`, `sau_pac`, `sau_remlot`, `sys_pes`).

**`/mnt/s/projetos/Receituario/JavaModel/web/Resources/`:**
- Purpose: Theme assets (CSS, images) and i18n string tables.
- Active theme: `SysmarTheme` (`client.cfg:59`).
- Active language: Portuguese (`client.cfg:57`).
- Note: `Resources/Portuguese/SysmarTheme/` is the actually-used pair; the GeneXusXEv2 and SimpleAndroid trees are GX defaults retained but not selected.

**`/mnt/s/projetos/Receituario/JavaModel/web/modules/` and `services/`:**
- Purpose: Staging of GeneXus-supplied runtime jars before they are bundled into `WEB-INF/lib/` at build time. Mirrors of `WEB-INF/lib/GeneXus.jar` and `WEB-INF/lib/GXWebSocket.jar` in the deployed copy.

**`/mnt/s/projetos/Receituario/JavaModel/web/drivers/`:**
- Purpose: Repository of all JDBC drivers GeneXus knows about. Only `postgresql-9.1-902.jdbc3.jar` is engaged at runtime per `client.cfg:113`.
- All drivers are copied to `WEB-INF/lib/` during deployment regardless of `DBMS` setting.

**`/mnt/s/projetos/Receituario/JavaModel/web/bootstrap/`, `HMask/`, `SlideDownMenu/`, `GxMask-master/`, `devmenu/`:**
- Purpose: Static JS/CSS vendor components used by generated forms. Exposed publicly under `/static/*` after deploy.

**`/mnt/c/.../ReceituarioJavaEnvironment/WEB-INF/classes/`:**
- Purpose: Tomcat classpath — every generated object compiled flat (no packages). 665 files.
- Contains: `<name>.class`, `<name>_impl.class`, `<name>__default.class`.
- `__default` companion holds SQL cursor declarations referenced by the object's impl (e.g. `acessamodulo__default.class` for `acessamodulo.java`).

**`/mnt/c/.../ReceituarioJavaEnvironment/WEB-INF/lib/`:**
- Purpose: 61 third-party jars (GeneXus runtime + commons + crypto + PDF + JSON).

**`/mnt/c/.../ReceituarioJavaEnvironment/static/`:**
- Purpose: Public asset root mapped by `client.cfg:46` (`WEB_IMAGE_DIR=/static`). Mirrors of `bootstrap/`, `Resources/`, etc.

**`/mnt/c/.../ReceituarioJavaEnvironment/PublicTempStorage/`:**
- Purpose: Runtime BLOB write target (`client.cfg:60` — `CS_BLOB_PATH=PublicTempStorage`). Files generated during PDF/report runs land here.

**`/mnt/c/.../ReceituarioJavaEnvironment/LayoutMetadata/`:**
- Purpose: Layout templates for `iText`-based reports/PDFs (`client.cfg:52` — `PRINT_LAYOUT_METADATA_DIR=LayoutMetadata`). Only present in the deployed copy.

## Key File Locations

**Entry Points (servlets):**
- `home.java` → `/servlet/home` — KB Start Object.
- `acessamodulo.java` → `/servlet/acessamodulo` — module-access gate.
- `cadastrarsenha.java`, `cadastrasenha1.java` → first-time password setup.
- `hmudasenhalogin.java` → change password.
- `hindex.java` → `/servlet/hindex` — landing page after login.

**Infrastructure servlets (configured in `web.xml`):**
- `/gxobject` (file uploads), `/oauth/*` (disabled stubs), `/gx_valid_service` (client validation), `/servlet/com.genexus.webpanels.GXResourceProvider` (resources).
- See `/mnt/s/projetos/Receituario/JavaModel/web/web.xml` lines 19-122.

**Configuration:**
- `client.cfg` — DB URL (line 114), credentials (lines 107-108, encoded), pool sizes (122-128), language/date format (57-58, 82-88), theme (59), cache (63-70), session encryption (62).
- `context.xml` — Tomcat context overrides (JarScanner restriction).
- `CloudServices.config` — WebNotifications service binding.
- `GeneXus.services` — exposable services manifest.
- `web.xml` — servlet/listener/filter wiring.
- `GXPRN.INI`, `PDFReport.ini` (deployed copy only) — print/PDF settings.

**Core domain code (active transactions, 35 detected):**
- Healthcare core: `sau_pac*` (paciente), `sau_pesf*` (profissional de saúde), `sau_pesf_pac*` (vínculo prof-paciente), `sau_pesf_profext*` (profissional externo).
- Prescription / dispensing: `sau_rem*` (remédio), `sau_remlot*` (lote), `sau_remobs*`, `sau_recesp*` (receita especial), `sau_aprrem*` (aprovação remédio), `sau_tiprem*` (tipo receita).
- Catalogs: `sau_esp*` (especialidade), `sau_imp*` (impressão), `sau_loc*` (local), `sau_uni*` (unidade), `sau_unisetor*`, `sau_bai*` (bairro), `sau_dis*` (distrito), `sau_concla*`, `sau_cep*`, `sau_cbor*` (CBOR ocupacional), `sau_cnae*`.
- Care groups: `sau_prg*` (programa), `sau_prggrp*` (grupo de programa), `sau_pro*` (procedimento), `sau_proesp*` (procedimento especialidade), `sau_paccns*` (CNS paciente), `sau_pacprn*` (prontuário).
- Professionals & doctor parameters: `sau_prf*`, `sau_prfcon*`, `sau_par*` (`sau_par_amb`, `sau_par_ger`), `sau_fun*` (função).
- Users / permissions: `sau_usu*`, `sau_usucon*`, `sau_usupar*`, `sau_usuuni*`, `sau_log*` (audit log).
- System-level: `sys_pes*` (pessoa — supertype), `sys_emp*` (empresa).

**Search / lookup procedures:**
- `buscaibge.java`, `buscaloginusuario.java`, `buscaprocedimentoprincipal.java`, `buscadocumentolgpd.java`, `buscaespecialidade.java`, `buscaquantidadesaldomedicamento.java`, `buscasetorsaldomedicamento.java`, `buscaunidadesaldomedicamento.java`, `buscarposologiaindividual.java`, `pfindtab.java`, `pgettabname.java`.

**Reports (HTTP procedures → PDF via `iText.jar`):**
- `rsau_recesp.java`, `rsau_recesp_pagint.java`, `rsau_pacextdet.java`, `rsau_pacextres.java`.

**Master pages:**
- `appmasterpage.java` — primary app shell.
- `rwdmasterpage.java`, `rwdpromptmasterpage.java`, `rwdrecentlinks.java` — responsive variants.
- `hmasterpagelogin.java`, `hmasterpageprompt.java`, `hmasterpagesuporte.java`, `hmasterpagetransaction.java`, `hmasterpageww.java`, `promptmasterpage.java`.

**SDTs (shared DTOs):**
- `SdtContext.java` (144 KB — environment context).
- `SdtSAU_PAC.java`, `SdtSAU_LOG.java`, `SdtSAU_REMLOT.java`, `SdtSYS_PES.java` — BC-shaped SDTs.
- `SdtSDT_INFO_PACIENTE_Item.java`, `SdtSDT_QTD_Item.java`, `SdtSDT_SAU_PESF_PAC.java`, `SdtSDT_SAU_REMLOT_Item.java`, `SdtSDT_TipoReceita_SDT_TipoReceitaItem.java`.
- `SdtSeguranca.java`, `SdtSlideDownMenuData_*`, `SdtTabOptions_*`, `SdtTransactionContext*`.

**Runtime stdlib (root, prefixed `Gx*`):**
- `GxWebStd.java` (238 KB — UI helpers), `GxObjectCollection.java`, `GxFullTextSearchReindexer.java`.

**Domain enums (`gxdomain*.java`):**
- `gxdomaintrnmode.java` (transaction mode INS/UPD/DLT), `gxdomainboolean90.java`, `gxdomaincores.java`, `gxdomainsus_rendafamemsalsmins.java`, `gxdomaintiposdecotas.java`, `gxdomaintiposdeprocs.java`, `gxdomainmenu.java`, `gxdomainnotorigem.java`, `gxdomainnotperfil.java`, `gxdomainpage.java`, `gxdomainparametro.java`, `gxdomaintabimagename.java`.

**Static assets (web-exposed at `/static/...`):**
- Path: `/mnt/c/Program Files/Apache Software Foundation/Tomcat 8.5/webapps/ReceituarioJavaEnvironment/static/`.
- Source: copies from `JavaModel/web/bootstrap/`, `devmenu/`, `HMask/`, `SlideDownMenu/`, `Resources/`, `Metadata/`, `GxMask-master/`.
- Per-object `.js` lives next to its `.java` (e.g. `sau_pac.js`, `hwwsau_pac.js`) and is served from the same path.

**Testing:**
- Not applicable. There are no test files. Generated GeneXus webapps have no unit-test layer; QA is done against the running KB in the IDE.

## Naming Conventions

### File-prefix conventions (GeneXus regen)

**Object stub + impl pair:**
- Every interactive object emits two files: `<name>.java` (thin servlet/callable stub) and `<name>_impl.java` (heavy logic). Example: `sau_pac.java` (48 lines, `extends GXWebObjectStub`) + `sau_pac_impl.java` (947 KB, `extends GXDataArea`).
- Procedures with no UI emit a single non-`_impl` file extending `GXProcedure`: `psau_inc_dis.java:14`.

**Object kind prefixes (verified by sampling):**

| Prefix | Kind | Example | Object type |
|--------|------|---------|-------------|
| `sau_*` (no h/p/r) | Transaction (form servlet) | `sau_pac.java` `sau_rem.java` `sau_pesf.java` | extends `GXWebObjectStub` |
| `<trn>_bc` | Business Component | `sau_pac_bc.java` (4 only) | extends `GXWebPanel implements IGxSilentTrn` |
| `<trn>_impl` | Generated impl body | `sau_pac_impl.java` | extends `GXDataArea` |
| `h*` | "Harmony" web object (web panel/master/section variant) | `home.java`, `hindex.java`, `hmasterpagelogin.java` | varies |
| `hww<trn>` | Work-With grid panel | `hwwsau_pac.java` (26 files) | servlet → grid impl |
| `hview<trn>` | View / read-only panel | `hviewsau_pac.java` (22 files) | servlet → view impl |
| `hprompt<trn>` | FK-lookup popup | `hpromptsau_pac.java` (58 files) | servlet → prompt impl |
| `hsau_<trn>general` | "General" section/tab embedded in transaction | `hsau_pacgeneral.java` (42 `hsau_*` files) | section |
| `p*` | Procedure (internal) | `psau_inc_dis.java` (56 `psau_*` + 14 others) | extends `GXProcedure` |
| `r*` | HTTP procedure / report | `rsau_recesp.java` (8 `rsau_*`) | extends `GXWebObjectStub`, "Program type: HTTP procedure" |
| `rwd*` | Responsive Web Design master page | `rwdmasterpage.java` | master page |
| `Sdt*` | Structured Data Type | `SdtSAU_PAC.java`, `SdtContext.java` (19 files) | value object |
| `Gx*` (capitalized) | Runtime stdlib | `GxWebStd.java`, `GxObjectCollection.java` | utility |
| `gxdomain*` | KB domain enum | `gxdomaintrnmode.java` (12 files) | enum-like class |
| `<name>__default` | SQL cursor companion (`.class` only, no `.java` in root) | `sau_pac__default.class` | runtime cursor decls |

**Note on `a*` / `dp*` prefixes:** The prompt mentioned `a*.java` for action classes and `dp*.java` for data providers as conventions to look for. They are not present in this codebase. The only files starting with lowercase `a` are `acessamodulo.java` and `appmasterpage.java` (both are normal objects, not "action"-prefixed). No `dp*.java` exists — data lookups are done by `busca*` procedures or inline in impls.

### Domain prefix layer (sub-namespace inside object names)

After the kind prefix (`h`, `p`, `r`, `sau_`, etc.) the next token names the **domain module** in the KB. These map directly to DB table prefixes.

| Domain prefix | Meaning (Portuguese) | Domain (English) | Example object | Example backing table |
|---------------|----------------------|------------------|----------------|----------------------|
| `sau_*` | Saúde | Health domain (umbrella) | `sau_pac.java` | `SAU_PAC` |
| `sau_pac` | Paciente | Patient | `sau_pac.java`, `Sdt_SAU_PAC.java` | `SAU_PAC` |
| `sau_pesf` | Pessoa Física (Profissional de Saúde) | Healthcare professional | `sau_pesf.java`, `hviewsau_pesf.java` | `SAU_PESF` |
| `sau_pro` | Procedimento | Procedure | `sau_pro.java`, `hwwsau_pro.java` | `SAU_PRO` |
| `sau_rem` | Remédio | Medicine | `sau_rem.java`, `psau_inc_med.java` | `SAU_REM` |
| `sau_remlot` | Lote de Remédio | Medicine batch | `sau_remlot_bc.java` | `SAU_REMLOT` |
| `sau_recesp` | Receita Especial | Special prescription | `sau_recesp.java`, `rsau_recesp.java` | `SAU_RECESP` |
| `sau_esp` | Especialidade | Specialty | `sau_esp.java`, `buscaespecialidade.java` | `SAU_ESP` |
| `sau_uni` | Unidade | Unit / facility | `sau_uni.java`, `sau_unisetor.java` | `SAU_UNI`, `SAU_UNISETOR` |
| `sau_vac` (referenced) | Vacina | Vaccine | — (not in current object list) | — |
| `sau_usu` | Usuário | User | `sau_usu.java`, `generatorsau_usu.java`, `psau_exc_usu.java` | `SAU_USU` |
| `sau_fun` | Função | Role / function | `sau_fun.java`, `hwwsau_fun.java` | `SAU_FUN` |
| `sau_log` | Log (auditoria) | Audit log | `sau_log.java`, `sau_log_bc.java` | `SAU_LOG` |
| `sys_pes` | Pessoa (sistema) | Person (system supertype) | `sys_pes.java`, `sys_pes_bc.java`, `SdtSYS_PES.java` | `SYS_PES` |
| `sys_emp` | Empresa | Company | `sys_emp.java` | `SYS_EMP` |
| `sys_mun` (referenced) | Município | Municipality | accessed by `buscaibge.java` | `SYS_MUN` (see `Metadata/TableAccess/BuscaIBGE.xml`) |

**How to find the table for an object:** read `Metadata/TableAccess/<ObjectName>.xml` — every object has one. Example `Metadata/TableAccess/AcessaModulo.xml` declares dependencies on `SAU_USU` and `SAU_USUUNI`.

### Casing rules

- Java filename: all-lowercase with underscores (`sau_pac.java`, `hwwsau_pac.java`, `psau_inc_dis.java`).
- KB object name (per file banner): UPPERCASE (`File: SAU_PAC`, `File: PSAU_INC_DIS`, `File: HWWSAU_PAC`).
- SDT class name: capitalized prefix `Sdt` + UPPERCASE object (`SdtSAU_PAC.java`).
- Runtime stdlib: capitalized `Gx` prefix (`GxWebStd`, `GxObjectCollection`).
- `__default` companion: double-underscore suffix, lowercase (`sau_pac__default.class`).

## Where to Add New Code

> Every file in this tree is regenerated from the GeneXus KB. There is no "add a new Java file" workflow. To add code, you add KB objects and regenerate.

**New Transaction (form-bound business object):**
- In GeneXus IDE: create Transaction `SAU_<NAME>` with attributes mapping to a new/existing table.
- Regen produces: `sau_<name>.java` + `sau_<name>_impl.java` in `/mnt/s/projetos/Receituario/JavaModel/web/`, plus `Metadata/TableAccess/Sau_<name>.xml`.
- If marked as Business Component: also produces `sau_<name>_bc.java` + `gxmetadata/sau_<name>.json`.
- If a Work-With pattern is applied: also produces `hwwsau_<name>.*`, `hviewsau_<name>.*`, `hpromptsau_<name>.*`, `hsau_<name>general.*`.

**New Procedure (server logic, no UI):**
- In GeneXus IDE: create Procedure `PSAU_<NAME>` (internal) or `RSAU_<NAME>` (HTTP/PDF).
- Regen produces: `psau_<name>.java` (or `rsau_<name>.java`) at the root, extending `GXProcedure` or `GXWebObjectStub` respectively.

**New Web Panel (custom screen, not tied to a single table):**
- In GeneXus IDE: create Web Panel `<NAME>` (e.g. `HOME`, `HLIBERACAO`).
- Regen produces: `<name>.java` + `<name>_impl.java` (e.g. `home.java`, `hliberacao.java`).

**New SDT (typed value object):**
- In GeneXus IDE: create SDT.
- Regen produces: `Sdt<NAME>.java`.

**New static asset:**
- Drop under `/mnt/s/projetos/Receituario/JavaModel/web/bootstrap/`, `HMask/`, `SlideDownMenu/`, or `Resources/SysmarTheme/`.
- Will be copied to deployed `static/` during build.

**New third-party jar:**
- Drop into `/mnt/s/projetos/Receituario/JavaModel/web/modules/` or `services/`.
- Will be copied to deployed `WEB-INF/lib/` during build.

**Changing servlet wiring (rare):**
- Edit `/mnt/s/projetos/Receituario/JavaModel/web/web.xml`. This file is also regenerated by the GeneXus environment configuration; preferred path is to change the wiring in the KB's environment settings.

**Changing DB connection / runtime config:**
- Edit `/mnt/s/projetos/Receituario/JavaModel/web/client.cfg` (regenerated from KB Environment).
- Deployed copy is `/mnt/c/.../ReceituarioJavaEnvironment/WEB-INF/classes/client.cfg` — needs the same edit (or redeploy).

## Special Directories

**`Metadata/TableAccess/`:**
- Purpose: Per-object table dependency manifest (one XML per object).
- Generated: Yes (by GeneXus). Committed: Yes.
- Example content (`AcessaModulo.xml`):
  ```xml
  <Dependencies type="guid" guid="35f99b7c-...">
    <Table name="SAU_USU" />
    <Table name="SAU_USUUNI" />
  </Dependencies>
  ```

**`gxmetadata/`:**
- Purpose: BC JSON schemas + GeneXus runtime version stamp.
- Generated: Yes. Committed: Yes.
- `gxversion.json` content: `{ "version":"1014253"}` — matches `client.cfg:54` (`GX_BUILD_NUMBER=118219`) and `client.cfg:70` (`CACHE_INVALIDATION_TOKEN=20265201014253`).
- One JSON per Business Component (only 4 BCs exist).

**`LayoutMetadata/` (deployed only):**
- Purpose: GeneXus print/PDF layout templates referenced by `rsau_*` HTTP procedures.
- Generated: Yes (deploy step). Committed in deployed copy.

**`PublicTempStorage/` (deployed only):**
- Purpose: Runtime BLOB output area for PDF/report results.
- Generated: At runtime. Should be writable by the Tomcat process.

**`PrivateTempStorage/` (referenced by `client.cfg:51`):**
- Purpose: Internal temp media buffer.
- Created on demand at deploy root.

**`Resources/Portuguese/SysmarTheme/`:**
- Purpose: Active theme + locale combination. UI strings + theme images.
- Generated: From KB i18n + Theme objects.

**`drivers/`:**
- Purpose: JDBC drivers for all supported DBMSs. All copied to `WEB-INF/lib/` but only the one matching `client.cfg:121` (`DBMS=postgresql`) is loaded at runtime via `JDBC_DRIVER=org.postgresql.Driver`.

---

*Structure analysis: 2026-05-21*
