# Technology Stack

**Analysis Date:** 2026-05-21

## Source-of-Truth Notice (Read First)

**This directory is generated output, not hand-written source code.** The authoritative artifact for this application is a **GeneXus 12 Knowledge Base** stored as a SQL Server database file on the design-time machine:

- KB file: `S:\projetos\Receituario\GX_KB_Receituario.mdf` (SQL Server / LocalDB metadata store)
- KB editor: GeneXus 12+ (Genexus Java Generator stamps say `15_0_7-118219` — see `Author:` headers in every `.java` file)
- Output target: this directory (`Receituario/JavaModel/web/`) is the "Java Model" regeneration produced by `GXJMake.exe` / `CALLMAKE.BAT`

**Implication:** Edits made directly to `.java`, `.js`, `.css`, or `.xml` files under `Receituario/JavaModel/web/` are **wiped on every regen**. Real changes must be made in the GeneXus IDE against the KB and then re-generated. The deployment to Tomcat is a separate copy step that mirrors this directory into the Tomcat `webapps/` directory.

**Two databases exist, do not confuse them:**

| Database | Engine | Purpose | How accessed |
|----------|--------|---------|--------------|
| GeneXus KB | SQL Server (`.mdf`) | Design-time metadata (transactions, web panels, procedures) | GeneXus IDE only |
| Runtime application DB | PostgreSQL | Live application data (`saude-mandaguari`) | JDBC from the running webapp — see `web/client.cfg` |

The runtime `client.cfg` (`Receituario/JavaModel/web/client.cfg` line 121 `DBMS=postgresql`, line 113-114 `JDBC_DRIVER=org.postgresql.Driver` / `DB_URL=jdbc:postgresql://127.0.0.1:5432/saude-mandaguari`) confirms that the deployed servlet container connects to PostgreSQL, not SQL Server. Every generated `.java` file also carries `Main DBMS: PostgreSQL` in its header (e.g. `web/acessamodulo.java:7`, `web/buscaibge.java:7`, `web/hindex.java:7`).

## Languages

**Primary:**
- Java (source 1.5/1.6 era idioms; uses `extends GXProcedure`, raw `java.sql.*`) — 468 `.java` files at `Receituario/JavaModel/web/*.java`. All files are **generated** (header `Author: GeneXus Java Generator version 15_0_7-118219`).
- JavaScript (ES3-compatible; jQuery 1.2.6 era) — 188 `.js` files at `Receituario/JavaModel/web/*.js`, e.g. `web/gxgral.js`, `web/gxprint.js`, `web/calendar.js`, plus per-object scripts like `web/hindex.js`, `web/cadastrasenha1.js`.

**Secondary:**
- CSS — 12 stylesheets at the web root (`web/autosuggest.css`, `web/calendar-system.css`, `web/container.css`) and theme stylesheets under `web/Resources/Portuguese/SysmarTheme.css`, `web/Resources/Portuguese/GeneXusXEv2.css`.
- SQL (PostgreSQL dialect) — embedded as JDBC string literals inside the generated Java `*.java` files (no standalone `.sql` files in the model output).
- XML — 154 files: per-object table-access metadata under `web/Metadata/TableAccess/*.xml`, layout metadata under `web/LayoutMetadata/` (Tomcat copy), and servlet descriptors at the web root (`web/web.xml`, `web/web6.xml`, `web/web6_5.xml`, `web/web7.xml`, `web/web7_5.xml`, plus `_native_ws` variants — different Servlet API levels GeneXus can emit; only `web/web.xml` is the active one used at deploy).

## Runtime

**Servlet container:**
- Apache Tomcat 8.5 (Windows install)
- Catalina base: `C:\Program Files\Apache Software Foundation\Tomcat 8.5`
- WSL mirror path: `/mnt/c/Program Files/Apache Software Foundation/Tomcat 8.5`
- Control scripts: `C:\Program Files\Apache Software Foundation\Tomcat 8.5\bin` (`startup.bat`, `shutdown.bat`, `catalina.bat`)
- Deployed webapp folder: `C:\Program Files\Apache Software Foundation\Tomcat 8.5\webapps\ReceituarioJavaEnvironment`
- Context URL prefix (inferred from folder name): `/ReceituarioJavaEnvironment`

**JVM:**
- `web/web.xml` declares Servlet API 3.0 (`web-app_3_0.xsd`, `version="3.0"`) — requires Java 7+ as host runtime. Generated source uses no Java-7+ language features (uses pre-generics style in many spots), so the bytecode target is JVM-compatible with Java 7/8 (Tomcat 8.5 minimum is Java 7).
- No `MANIFEST.MF` Build-Jdk hint inside the generated tree (the only `META-INF` directory in the deploy is `Receituario/JavaModel/web/META-INF/` which is empty of metadata).

**Build tool:**
- GeneXus-supplied native builder `GXJMake.exe` (`Receituario/JavaModel/web/GXJMake.exe`), driven by `Receituario/JavaModel/web/CALLMAKE.BAT`.
- No Maven/Gradle/Ant build script for application code (the bundled `ant.jar`, `ant-nodeps.jar`, `ant-jakarta-bcel.jar`, `bcel-5.1.jar`, `asm-3.1.jar` at the web root are GeneXus internal helpers used by the compiler step, not project-level build files).

**Package manager:**
- None. Dependencies are vendored as `.jar` files into `Receituario/JavaModel/web/*.jar` (62 jars at the model root, 61 jars in the Tomcat `WEB-INF/lib/`). There is no `pom.xml`, `build.gradle`, `ivy.xml`, or lockfile anywhere in the tree.

## GeneXus Version Markers

| File | Meaning |
|------|---------|
| `web/GJS118219.VER` | GeneXus Java Generator build `118219` (matches `client.cfg` line 54 `GX_BUILD_NUMBER=118219` and the `Author: GeneXus Java Generator version 15_0_7-118219` header stamped into every `.java` file) |
| `web/GX_PRO12.002` | GeneXus "Pro 12" model marker (54 KB binary) — confirms GeneXus 12 lineage |
| `web/GXHPRO12.002`, `web/GXPPRO12.002`, `web/GXSPRO12.002`, `web/GXTPRO12.002`, `web/GXYPRO12.002` | Per-object-class GeneXus 12 generator stamp files |
| `web/GxMask.1.0.2.VER` | Bundled GxMask user control v1.0.2 (sources under `web/GxMask-master/`) |
| `web/HMask.0.47.VER` | Bundled HMask user control v0.47 (sources under `web/HMask/`, ships `jquery-1.2.6.min.js` and `jquery.meiomask.compressed.js`) |
| `web/SlideDownMenu.2.0.VER` | Bundled SlideDownMenu user control v2.0 (sources under `web/SlideDownMenu/`) |
| `web/gxmetadata/gxversion.json` | `{ "version":"1014253"}` — KB version stamp (also surfaces as `CACHE_INVALIDATION_TOKEN=20265201014253` in `web/client.cfg:70` and `VER_STAMP=20260519.191424` line 81) |

## Frameworks

**Core (GeneXus runtime, vendored):**
- `GeneXus.jar` (`WEB-INF/lib/GeneXus.jar`) — main GX runtime: `com.genexus.*`, `com.genexus.db.*`, `com.genexus.webpanels.*`, `com.genexus.internet.*`, `com.genexus.xml.*`, `com.genexus.search.*`, `com.genexus.filters.ExpiresFilter`.
- `gxclassR.jar` (`WEB-INF/lib/gxclassR.jar`) — GeneXus reports runtime.
- `GxUtils.jar` (`web/GxUtils.jar`) — utility helpers used by the generated code.
- `GXWebSocket.jar` (`WEB-INF/lib/GXWebSocket.jar`) — WebSocket-based Web Notifications service (`com.genexus.internet.websocket.GXWebSocket`, referenced from `web/CloudServices.config`).

**REST/JAX-RS stack (vendored Jersey 2.22.2):**
- `WEB-INF/lib/jersey-server.jar`, `jersey-client.jar`, `jersey-common.jar`, `jersey-container-servlet-core.jar`, `jersey-entity-filtering-2.22.2.jar`, `jersey-guava-2.22.2.jar`, `jersey-media-json-jackson-2.22.2.jar`
- HK2 DI: `hk2-api-2.4.0-b34.jar`, `hk2-locator-2.4.0-b34.jar`, `hk2-utils-2.4.0-b34.jar`, `javax.inject-2.4.0-b34.jar`
- JAX-RS API: `javax.ws.rs-api-2.0.1.jar`
- The active `web/web.xml` (and Tomcat copy `WEB-INF/web.xml`) does **not** register Jersey's `ServletContainer` — only GeneXus-internal servlets (see Integrations). The alternate `web/web6.xml`, `web/web6_5.xml`, etc. do register `org.glassfish.jersey.servlet.ServletContainer` named `JerseyListener` and could be swapped in if the KB regenerates with REST services enabled.

**JSON:**
- Jackson 2.7.3 (`jackson-core-2.7.3.jar`, `jackson-databind-2.7.3.jar`, `jackson-annotations-2.7.3.jar`, `jackson-jaxrs-json-provider-2.7.3.jar`, `jackson-module-jaxb-annotations-2.7.3.jar`).

**XML / SOAP:**
- `xercesImpl.jar`, `xml-apis-1.4.01.jar` (XML parsing).
- `xmlsec.jar` (XML signature/encryption, used by SAML/WS-Security-style payloads).
- `dom4j-1.6.1.jar`, `jdom-2.0.0.jar` (DOM helpers used by `com.genexus.xml.*`).
- `Tidy.jar` (HTML/XML cleanup used by some GX runtime parsers).
- The `web/web6_native_ws.xml` / `web/web7_5_native_ws.xml` variants register `com.sun.xml.ws.transport.http.servlet.WSServlet` (JAX-WS) — not active in the deployed `web.xml`, but available if the KB exposes SOAP services.

**Office/PDF reporting:**
- `iText.jar` + `iTextAsian.jar` — PDF generation used by GX reports (config in `WEB-INF/PDFReport.ini`).
- `poi.jar`, `poi-ooxml.jar`, `poi-ooxml-schemas.jar`, `poi-scratchpad.jar`, `xmlbeans.jar` — Apache POI for Excel/Word output.
- `noggit-0.5.jar` — JSON parser used by POI/PDF pipelines.
- `web/printingappletsigned.jar` (306 KB, signed) — legacy Java-applet-based client-side print component. Indicates the app was originally designed to push reports through a browser-side applet (requires NPAPI / older JRE on the client); see `INTEGRATIONS.md`.
- `web/gxprintserver.jar` — server-side print helper bundled by GeneXus.
- `web/Resources/Portuguese/GXPRN.INI` and `web/WEB-INF/GXPRN.INI` — printer defaults (paper, scale, duplex).
- `web/WEB-INF/PDFReport.ini` — global PDF rendering config (margins, font search paths, `FontsLocation= C:\Program Files\Apache Software Foundation\Tomcat 8.5;`).

**Full-text search:**
- Lucene 2.2.0 (`lucene-core-2.2.0.jar`, `lucene-highlighter-2.2.0.jar`, `lucene-spellchecker-2.2.0.jar`) plus `spatial4j-0.6.jar`, `jts-1.14.jar`, `jtsio-1.14.jar` — used by `com.genexus.search.*` (imported by virtually every generated `.java` file). `web/GxFullTextSearchReindexer.java` is the reindexer entry point.

**Crypto:**
- Bouncy Castle 1.47 (`bcpkix-jdk15on-147.jar`, `bcprov-jdk15on-147.jar`) — used for credential encryption (note `USER_ID=qVxDieU5ngJ+7cCSaUXFYk==` and `USER_PASSWORD=...` in `web/client.cfg` are GeneXus-encrypted credential blobs, not plaintext).
- `xmlsec.jar` for XML-DSig.

**Networking / mail / FTP:**
- `commons-net-3.3.jar` (FTP/SMTP/Telnet).
- `mail.jar`, `activation.jar` (JavaMail 1.4-era — SMTP send capability via `com.genexus.internet`).

**JDBC drivers (vendored, multi-DBMS — GeneXus ships them all regardless of active DBMS):**
- `postgresql-9.1-902.jdbc3.jar` — **active driver** per `web/client.cfg:113` (`JDBC_DRIVER=org.postgresql.Driver`)
- `jtds-1.2.jar` (SQL Server / Sybase)
- `mysql-connector-java-5.1.11-bin.jar`
- `ojdbc6.jar` (Oracle)
- `sqlitejdbc-v056.jar` (SQLite)
- `jt400.jar` (IBM i / DB2 for i)

**Apache Commons:**
- `commons-fileupload-1.3.2.jar`, `commons-io-2.2.jar`, `commons-lang-2.4.jar`, `commons-logging.jar`.

**Logging:**
- `log4j-1.2.15.jar` (no `log4j.properties` discovered in the deploy — likely uses GeneXus defaults baked into `GeneXus.jar`).

**Validation:**
- `validation-api-1.1.0.Final.jar` (Bean Validation API; no Hibernate Validator impl bundled — declarative only).

**Date/time:**
- `joda-time-2.8.2.jar` (used by `com.genexus.GXutil`).

**Tomcat API stubs (compile-time only — must not be loaded by Tomcat):**
- `annotations-api.jar`, `servlet-api.jar` — present in the model root `Receituario/JavaModel/web/` for the `GXJMake` compile step. They are correctly **omitted** from the deployed `WEB-INF/lib/` (where Tomcat 8.5's own copies are used).

**Frontend libraries:**
- jQuery 1.2.6 (`web/HMask/jquery-1.2.6.min.js`) — ancient; used by HMask input-mask control.
- jQuery meioMask plugin (`web/HMask/jquery.meiomask.compressed.js`).
- Custom GeneXus runtime: `web/gxgral.js` (main client runtime), `web/gxgral.src.js` (debug build), `web/gxdec.js` (decimal formatting), `web/gxprint.js` (print bridge to the signed applet), `web/gxtimezone.js`.
- Calendar widget: `web/calendar.js` + 9 locale files (`calendar-pt.js`, `calendar-en.js`, `calendar-es.js`, `calendar-de.js`, `calendar-it.js`, `calendar-ja.js`, `calendar-zh.js`, `calendar-ct.js`, `calendar-setup.js`).
- Per-page generated JS (one `*.js` per web panel/transaction, e.g. `web/appmasterpage.js`, `web/hindex.js`, `web/cadastrasenha1.js`).

**Themes:**
- Active theme: `SysmarTheme` — set in `web/client.cfg:59` (`Theme=SysmarTheme`).
- Theme assets: `web/Resources/SysmarTheme/` (gif/png chrome) and stylesheet `web/Resources/Portuguese/SysmarTheme.css`.
- Fallback/legacy theme: `GeneXusXEv2` — assets under `web/Resources/GeneXusXEv2/`, stylesheet `web/Resources/Portuguese/GeneXusXEv2.css`.
- Mobile/SD theme bundled but not the active web theme: `SimpleAndroid` — `web/Resources/Portuguese/SimpleAndroid.json` and `web/Resources/SimpleAndroid/`.

## Key Configuration Files

| File | Role |
|------|------|
| `Receituario/JavaModel/web/client.cfg` | The single most important runtime config — language, theme, DBMS, JDBC URL, encrypted credentials, encryption mode, calendar/decimal formatting, build number. Mirror deployed at `Tomcat 8.5/webapps/ReceituarioJavaEnvironment/WEB-INF/classes/client.cfg`. |
| `Receituario/JavaModel/web/web.xml` | Active servlet descriptor. Servlet API 3.0. Registers GeneXus servlets only (no Jersey). Mirror at `Tomcat 8.5/webapps/ReceituarioJavaEnvironment/WEB-INF/web.xml`. |
| `Receituario/JavaModel/web/context.xml` | Tomcat per-app context. Sets `reloadable="true"`, disables resource caching, restricts JarScanner to `GXWebSocket.jar` only. Mirror at `Tomcat 8.5/webapps/ReceituarioJavaEnvironment/META-INF/context.xml`. |
| `Receituario/JavaModel/web/contextGXJarScanner.xml` | Alternate context that swaps in GeneXus's own `com.genexus.webpanels.GXJarScanner`. |
| `Receituario/JavaModel/web/contextScanFilter.xml` | Alternate context using a JarScanFilter approach. |
| `Receituario/JavaModel/web/GeneXus.services` | Exposes the offline event replicator service `com.genexuscore.genexus.sd.synchronization.offlineeventreplicator` (`GeneXus.SD.Synchronization.OfflineEventReplicator`). Indicates the KB has Smart Devices / offline-sync capability even though no `sd_*` Java files are emitted in the current Java Model regen. |
| `Receituario/JavaModel/web/CloudServices.config` | Wires the WebNotifications channel to `com.genexus.internet.websocket.GXWebSocket` (in-process WebSocket). All four handler properties (received / onopen / onclose / onerror) are empty. Mirror at `WEB-INF/CloudServices.config`. |
| `Receituario/JavaModel/web/WEB-INF/PDFReport.ini` (Tomcat copy) | Global PDF settings: margins, font location, embed-fonts, server-printing off, barcode-128 as image. |
| `Receituario/JavaModel/web/WEB-INF/GXPRN.INI` (Tomcat copy) | Default print dialog settings. |
| `Receituario/JavaModel/web/gxmetadata/gxversion.json` | KB version stamp. |
| `Receituario/JavaModel/web/gxmetadata/sau_*.json`, `sys_pes.json` | Per-transaction metadata for the main business entities (`sau_log`, `sau_pac`, `sau_remlot`, `sys_pes`). |
| `Receituario/JavaModel/web/Metadata/TableAccess/*.xml` | Per-procedure declared table access — useful as the inventory of which generated programs touch which tables. |
| `Receituario/JavaModel/web/GXcfg.java` | Bootstrap class referenced by `web.xml` via `<context-param>gxcfg</context-param>` — see `web.xml:124-127`. Acts as the GeneXus configuration entry point. |
| `Receituario/JavaModel/web/CALLMAKE.BAT` | Wrapper that invokes `GXJMake.exe` to recompile the model. |
| `Receituario/JavaModel/web/LastCallTree.info` | Generator log (skip per scope). |

## Generated Code Conventions (because there is no "src/" structure)

All Java classes live flat at the web root in the default package (no `package` statement; `Receituario/JavaModel/web/*.java`). Naming patterns observed in the generated output:

- `<name>.java` — the public callable class (e.g. `web/acessamodulo.java` for procedure `AcessaModulo`).
- `<name>_impl.java` — companion implementation for web panels (e.g. `web/hindex.java` + `web/hindex_impl.java`).
- `<name>__default.java` (double underscore) — the per-DBMS data-cursor strategy class compiled alongside the callable (e.g. `web/acessamodulo__default.class`).
- `h<name>` prefix — web panel ("HTML" objects, e.g. `web/hindex.java`, `web/hmasterpagelogin.java`, `web/hpromptsau_rem_receita.java`).
- `Sdt<Name>.java` / `StructSdt<Name>.java` — Structured Data Types (DTOs) and their backing struct (e.g. `web/SdtSAU_PAC.java`, `web/StructSdtSAU_PAC.java`).
- `appmasterpage*`, `rwdmasterpage*`, `rwdpromptmasterpage*`, `promptmasterpage*` — master page templates (responsive web design variants are emitted with `rwd` prefix).

## Platform Requirements

**Development (KB editing):**
- Windows host with GeneXus 12+ IDE installed.
- SQL Server / LocalDB attached to `S:\projetos\Receituario\GX_KB_Receituario.mdf` (user `sysmar`; password held outside this repo — do not commit).
- GeneXus regenerates Java output into `Receituario/JavaModel/web/`.

**Deployment / runtime:**
- Windows + Tomcat 8.5 at `C:\Program Files\Apache Software Foundation\Tomcat 8.5`.
- JRE 7 or 8 (Tomcat 8.5 minimum; generated bytecode is pre-Java-7-language-features and works on any 1.7+ JVM).
- PostgreSQL server reachable at `127.0.0.1:5432`, database `saude-mandaguari`, role per encrypted `USER_ID` in `web/client.cfg:107` (decrypted by GeneXus runtime at boot).
- Filesystem write access for `web/PublicTempStorage/` (BLOB cache, see `client.cfg:60 CS_BLOB_PATH=PublicTempStorage`) and `web/PrivateTempStorage/` (`client.cfg:51 TMPMEDIA_DIR=PrivateTempStorage`).
- Optional native: `web/GXDIB32.DLL`, `web/rbuildj.dll` — Windows-only native helpers (image/report rendering). The app will boot without them but will degrade some print/image functions.
- For full-fidelity printing, an end-user browser able to load `web/printingappletsigned.jar` (signed Java applet, NPAPI — effectively unsupported in any post-2018 mainstream browser).

---

*Stack analysis: 2026-05-21*
