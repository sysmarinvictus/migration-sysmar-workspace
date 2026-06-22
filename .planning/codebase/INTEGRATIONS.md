# External Integrations

**Analysis Date:** 2026-05-21

## Source-of-Truth Reminder

All integrations below are wired by the **GeneXus 12 Knowledge Base** stored as a SQL Server `.mdf` at `S:\projetos\Receituario\GX_KB_Receituario.mdf` and regenerated into `Receituario/JavaModel/web/`. The runtime files documented here describe what is wired in the current generation output, not what the KB editor might be reconfigured to emit on next regen. Edits to add/remove integrations must be done in the GeneXus IDE, not in the generated `.java`/`.xml` files.

## Databases

### Application runtime DB — PostgreSQL

- **Engine:** PostgreSQL 9.1+ (driver `WEB-INF/lib/postgresql-9.1-902.jdbc3.jar`, JDBC3 era — predates JDBC 4 auto-loading and uses the deprecated `org.postgresql.Driver` registration).
- **Host / port / database:** `jdbc:postgresql://127.0.0.1:5432/saude-mandaguari` (see `Receituario/JavaModel/web/client.cfg:114`, mirror `Tomcat 8.5/webapps/ReceituarioJavaEnvironment/WEB-INF/classes/client.cfg:114`).
- **DBMS marker in code:** every generated `.java` file carries `Main DBMS: PostgreSQL` in its header (e.g. `web/acessamodulo.java:7`, `web/buscaibge.java:7`, `web/hindex.java:7`, `web/SoapParm.java:7`, `web/imprimicomprador.java:7`).
- **Credentials:** GeneXus-encrypted blobs in `client.cfg:107-108` (`USER_ID=qVxDieU5ngJ+7cCSaUXFYk==`, `USER_PASSWORD=dudhggSQtliZfx4F86JMacC8tMkXXXE9WGMHMnI+K5w=`). Decrypted at boot by Bouncy Castle (`WEB-INF/lib/bcprov-jdk15on-147.jar`) using a GeneXus master key. The decrypted role is referred to as **`sysmar`** in operational notes; the actual password is intentionally not written to disk in this map.
- **Connection pool:** GeneXus-managed RW pool — `client.cfg:122-128` (`PoolRWEnabled=1`, `POOLSIZE_RW=10`, `RecycleRW=1`, `RecycleRWMin=30`). No external pool (HikariCP / DBCP) is bundled.
- **Isolation:** `ISOLATION_LEVEL=CR` (`client.cfg:119`, GeneXus shorthand for "cursor stability / read committed").
- **Connection timeout:** `CONN_TIMEOUT=300` (`client.cfg:15`).
- **Schema discovery:** see `Receituario/JavaModel/web/Metadata/TableAccess/*.xml` — one XML per procedure listing the tables it reads/writes. Inventory shows business tables under the `sau_*` prefix (e.g. `SAU_PAC` patients, `SAU_LOG` audit log, `SAU_REMLOT` prescription lots, `SAU_REM_RECEITA` prescriptions) plus a `sys_*` security set (`SYS_PES` people/users) and IBGE reference data.
- **Other JDBC drivers vendored but unused:** `jtds-1.2.jar` (SQL Server), `mysql-connector-java-5.1.11-bin.jar`, `ojdbc6.jar` (Oracle), `sqlitejdbc-v056.jar`, `jt400.jar` (IBM i). GeneXus ships them for portability; do not load-test against them — only PostgreSQL is selected at generation time.

### Design-time DB — SQL Server (GeneXus KB)

- **Engine:** SQL Server / LocalDB.
- **File:** `S:\projetos\Receituario\GX_KB_Receituario.mdf`.
- **Schema/user:** `sysmar`.
- **Purpose:** stores the GeneXus Knowledge Base (transactions, web panels, procedures, themes, theme styles). **Not** accessed by the running Tomcat application.
- **Access tool:** GeneXus 12+ IDE only.

## HTTP / Web endpoints exposed by the running webapp

Inferred from `Receituario/JavaModel/web/web.xml` (active descriptor) and the per-procedure / per-web-panel `.java` files. The webapp publishes three kinds of HTTP endpoints:

### 1. GeneXus infrastructure servlets (from `web.xml`)

| URL path (relative to `/ReceituarioJavaEnvironment/`) | Servlet class | Purpose |
|---|---|---|
| `/gxobject` | `com.genexus.webpanels.GXObjectUploadServices` | File upload sink for `<input type=file>` fields. |
| `/oauth/access_token` | `com.genexus.webpanels.GXOAuthAccessTokenDummy` | OAuth2 token endpoint — **wired to "Dummy" implementation**, meaning OAuth is registered but not actually enabled in this KB. |
| `/oauth/logout` | `com.genexus.webpanels.GXOAuthLogoutDummy` | OAuth logout (dummy). |
| `/oauth/userinfo` | `com.genexus.webpanels.GXOAuthUserInfoDummy` | OAuth userinfo (dummy). |
| `/gx_valid_service` | `com.genexus.webpanels.GXValidService` | Per-attribute server-side validation AJAX endpoint. |
| `/servlet/com.genexus.webpanels.GXResourceProvider` | `com.genexus.webpanels.GXResourceProvider` | Streams static resources (theme images, CSS) out of `Resources/` / `Resources/SysmarTheme/`. |
| `/oauth/gam/signin` | `com.genexus.webpanels.agamextauthinputDummy` | GAM (GeneXus Access Manager) OAuth signin — **dummy**, GAM is not active. |
| `/oauth/gam/callback` | `agamextauthinputDummy` | GAM callback — dummy. |
| `/oauth/gam/access_token` | `agamoauth20getaccesstokenDummy` | GAM token — dummy. |
| `/oauth/gam/userinfo` | `agamoauth20getuserinfoDummy` | GAM userinfo — dummy. |
| `/oauth/gam/signout` | `agamextauthinputDummy` | GAM signout — dummy. |
| `/servlet/com.genexus.webpanels.gxver` | `com.genexus.webpanels.gxver` | Returns GeneXus runtime version string — diagnostics. |

A single filter is mounted globally: `com.genexus.filters.ExpiresFilter` mapped to `/*` (`web.xml:129-149`), setting `Cache-Control` for images, `text/css`, and `application/javascript` to "access plus 36 hours".

Two listeners run at startup: `com.genexus.webpanels.ServletEventListener` and `com.genexus.webpanels.SessionEventListener` (`web.xml:11-17`).

### 2. Generated web panels (HTML over HTTP — the actual UI)

Each web panel is compiled to a class with `Program type: Main program` in its header and is reachable as `/<classname>.aspx`-style GeneXus URL (GeneXus serves them through its own dispatching inside `com.genexus.webpanels`, not via `<servlet-mapping>` entries — the framework auto-routes by class name). Names in this KB (sample, not exhaustive):

- Authentication & session: `web/hmasterpagelogin.java`, `web/cadastrarsenha.java`, `web/cadastrasenha1.java`, `web/bloqueiausuario.java`, `web/desbloqueiausuario.java`, `web/buscaloginusuario.java`.
- Home / shell: `web/hindex.java` (header: `wSysSAU - Sistema de Gestão da Saúde Pública Municipal`), `web/home.java`, `web/appmasterpage.java`, `web/rwdmasterpage.java`, `web/rwdpromptmasterpage.java`.
- Domain prompts/workflows: `web/hpromptsau_rem_receita.java` (prompts for receita/prescription), `web/buscaespecialidade.java`, `web/buscaibge.java`, `web/buscaprocedimentoprincipal.java`, `web/buscaquantidadesaldomedicamento.java`, `web/buscasetorsaldomedicamento.java`, `web/buscaunidadesaldomedicamento.java`, `web/buscarposologiaindividual.java`.
- Reports / printing: `web/imprimicomprador.java` (header: `Imprimi Comprador`).
- Master pages: `web/hmasterpageprompt.java`, `web/hmasterpagesuporte.java`, `web/hmasterpagetransaction.java`, `web/hmasterpageww.java`, `web/promptmasterpage.java`.

### 3. REST / SOAP (currently latent)

**REST (Jersey):** all Jersey 2.22.2 jars are present in `WEB-INF/lib/`, **but** the active `web/web.xml` does **not** register Jersey's `org.glassfish.jersey.servlet.ServletContainer`. The alternative emitted descriptors `web/web6.xml` (lines 15-38), `web/web6_5.xml`, `web/web7.xml`, `web/web7_5.xml` do register Jersey under servlet name `JerseyListener` with `javax.ws.rs.Application = #1GXApplication` and GZIP filters. To switch the deployment to REST mode, the KB would need to be regenerated such that one of these `.xml` files is promoted to `web.xml` (or the deploy step swapped). **No `ws_*.java`, `sd_*.java`, `api*.java`, or `rest*.java` files exist** at the web root, confirming this KB does not currently expose any custom REST endpoints.

**SOAP / JAX-WS:** `web/web6_native_ws.xml`, `web/web6_5_native_ws.xml`, `web/web7_native_ws.xml`, `web/web7_5_native_ws.xml` reference `com.sun.xml.ws.transport.http.servlet.WSServlet` (`GXServices` servlet name) for native web services. Not active in `web/web.xml`. The only SOAP-adjacent code in the generated tree is `web/SoapParm.java` — a generic XML-section reader for SOAP envelope parameters (`oReader.getName() == "host"` etc.), used by `com.genexus.internet.Location`. There are no `@WebService`-annotated classes among the 468 `.java` files.

**OAuth2 / GAM:** OAuth and GAM endpoints are registered with `*Dummy` implementations — the runtime returns stubs. The KB has the OAuth/GAM scaffolding wired but the providers are turned off. To enable, the KB must regenerate with non-dummy classes (`GXOAuthAccessToken` without the `Dummy` suffix).

## Web Notifications / Real-time

- **Provider:** `INPROCESS` channel (`web/CloudServices.config` / `WEB-INF/CloudServices.config`).
- **Implementation:** `com.genexus.internet.websocket.GXWebSocket` (bundled in `WEB-INF/lib/GXWebSocket.jar`).
- **Status:** registered but no handlers wired — all four properties (`WEBNOTIFICATIONS_RECEIVED_HANDLER`, `_ONOPEN_`, `_ONCLOSE_`, `_ONERROR_`) are empty strings. Effectively dormant.
- **Special handling:** `web/context.xml` / `META-INF/context.xml` explicitly pluggability-scans only `GXWebSocket.jar` (`<JarScanFilter pluggabilitySkip="*" tldSkip="*" pluggabilityScan="GXWebSocket.jar"/>`) — this is the WebSocket TLD that must be picked up at boot.

## Offline / Smart Devices Sync

- **Service declaration:** `Receituario/JavaModel/web/GeneXus.services` exposes `GeneXus.SD.Synchronization.OfflineEventReplicator` → `com.genexuscore.genexus.sd.synchronization.offlineeventreplicator`.
- **Status:** the service is declared (so a Smart Devices client could call it), but the current Java Model regen contains no `sd_*` Java files at the web root. The mobile/offline application is generated separately (likely a sibling `JavaSDApp` / `AndroidModel` folder in the KB outputs — not in scope of this map).
- **Theme readiness:** `web/Resources/Portuguese/SimpleAndroid.json` and `web/Resources/SimpleAndroid/` are present, again indicating SD generation is configured in the KB.

## Authentication & Identity

- **Mode:** application-managed login (custom). The KB ships its own user/role tables (`SYS_PES`, `SAU_USU`) and login pages (`web/hmasterpagelogin.java`, `web/cadastrarsenha.java`, `web/cadastrasenha1.java`, `web/bloqueiausuario.java`, `web/desbloqueiausuario.java`).
- **GAM (GeneXus Access Manager):** **NOT enabled** — all GAM servlets in `web/web.xml:46-65` point to `*Dummy` classes (`agamextauthinputDummy`, `agamoauth20getaccesstokenDummy`, `agamoauth20getuserinfoDummy`). The KB has the GAM module imported but its providers are not active in this generation.
- **LDAP:** disabled — `client.cfg:3 LDAP_LOGIN=0`, `LDAP_HOST=` empty, `LDAP_PRINCIPAL=` empty.
- **Integrated Windows security:** disabled — `client.cfg:78 EnableIntegratedSecurity=0`.
- **Session encryption:** `client.cfg:62 USE_ENCRYPTION=SESSION` — session IDs and URL parameters are AES-encrypted by the GeneXus runtime (`com.genexus.GXKey` / `com.genexus.cryptography.*` from `GeneXus.jar`, with Bouncy Castle providers).
- **Password storage / verification:** handled inside generated procedures (`web/buscaloginusuario.java`, `web/cadastrasenha1.java`) reading from `SYS_PES`/`SAU_USU` tables on PostgreSQL.

## Email / SMS

- **SMTP library:** `WEB-INF/lib/mail.jar` + `activation.jar` (JavaMail) is on the classpath, and the GX runtime exposes `com.genexus.internet.GXSMTPSession` via `GeneXus.jar`.
- **SMTP host config:** `client.cfg:10 SMTP_HOST=` — **empty**. No active SMTP target.
- **In generated code:** no procedure imports a mail class directly (the matches for `SMTP` / `sendMail` in `*.java` are coincidental — e.g. the `_impl.java` matches are for unrelated string content). Email send capability exists in the runtime but is not used by any currently emitted procedure.
- **SMS:** no SMS provider configured. No Twilio / Nexmo / Zenvia / GeneXus-Cloud-SMS imports in the source.

## File Storage

- **Public temp storage** (e.g. user-downloadable PDFs, blobs returned to the browser):
  - Path: `Receituario/JavaModel/web/PublicTempStorage/` (`client.cfg:60 CS_BLOB_PATH=PublicTempStorage`).
  - Lifecycle: GeneXus runtime writes generated PDFs / Excel reports here and serves them via short-lived URLs.
- **Private temp storage** (server-side scratch):
  - Path implied by `client.cfg:51 TMPMEDIA_DIR=PrivateTempStorage` (folder created on demand under the webapp).
- **Layout metadata** (print layouts):
  - `client.cfg:52 PRINT_LAYOUT_METADATA_DIR=LayoutMetadata` → `Receituario/JavaModel/web/LayoutMetadata/` (deployed copy).
- **Resource provider** for theme/UI assets:
  - `Receituario/JavaModel/web/Resources/` served via the `GXResourceProvider` servlet.
- **Image directory** (legacy static):
  - `client.cfg:46 WEB_IMAGE_DIR=/static` → `Receituario/JavaModel/web/static/` (currently empty — images flow through `Resources/` instead).
- **No cloud storage** (S3 / Azure Blob / GCS) is configured. No AWS SDK / Azure SDK jars present in `WEB-INF/lib/`.

## Full-Text Search

- **Engine:** Apache Lucene 2.2.0 (`WEB-INF/lib/lucene-core-2.2.0.jar`, `lucene-highlighter-2.2.0.jar`, `lucene-spellchecker-2.2.0.jar`) plus Spatial4j / JTS.
- **GeneXus glue:** `com.genexus.search.*`, imported by every generated `.java` file (`import com.genexus.search.*;` at line 11 of e.g. `web/buscaibge.java`).
- **Index location / reindex:** `Receituario/JavaModel/web/GxFullTextSearchReindexer.java` is the manual reindex entry point. Index folder is GeneXus-runtime-managed under the webapp working directory.
- **No Solr / Elasticsearch** — purely embedded Lucene.

## Reporting / Printing

- **PDF reports:** server-side rendering with iText (`WEB-INF/lib/iText.jar`, `iTextAsian.jar`).
  - Configured by `WEB-INF/PDFReport.ini`: 0.75-inch top/left margins, 6-pt bottom margin, no embed-fonts, `Barcode128AsImage= true`, `ServerPrinting= false`.
  - Font search path: `C:\Program Files\Apache Software Foundation\Tomcat 8.5;` (i.e. it scans the Tomcat install dir for `.ttf` fonts) plus `c:\windows\fonts\micross.ttf` and `c:\windows\fonts\tahoma.ttf` (explicit mapping).
- **Excel/Word output:** Apache POI 3.x (`poi.jar`, `poi-ooxml.jar`, `poi-ooxml-schemas.jar`, `poi-scratchpad.jar`, `xmlbeans.jar`).
- **Native Windows printer dialog:** `web/GXDIB32.DLL` and `web/rbuildj.dll` are Windows-only native helpers loaded by `gxprintserver.jar` for direct-to-printer paths.
- **Client-side print bridge (legacy):** `Receituario/JavaModel/web/printingappletsigned.jar` is a code-signed Java applet that the GeneXus runtime injects via `web/gxprint.js` to enable browser-side direct printing.
  - **Compatibility risk:** signed applets require NPAPI; effectively dead in Chrome 45+ (2015), Firefox 52+ (2017), and all current Edge. Users on a modern browser will not be able to use applet-based print and will fall back to PDF download.
- **Print defaults:** `WEB-INF/GXPRN.INI` — paper-size 256 (Windows DMPAPER constant), portrait/landscape via `Orientation=2`, single copy.
- **Sample print procedure:** `web/imprimicomprador.java` (`Description: Imprimi Comprador`).

## HTTP Client (outbound)

- **API:** `com.genexus.internet.HttpClient` is available in `GeneXus.jar`, but no generated procedure in the current Java Model regen actually instantiates it (no source-file matches for `new HttpClient` / `GxHttpClient` outside of GeneXus runtime code).
- **Conclusion:** the application does not call any external REST APIs from server-side procedures in this generation. Outbound HTTP capability exists but is unused.
- **FTP:** `commons-net-3.3.jar` is on the classpath (the GX runtime can do FTP/SMTP/Telnet), but no procedure imports it.

## Monitoring & Observability

- **Logging library:** Log4j 1.2.15 (`WEB-INF/lib/log4j-1.2.15.jar`).
- **Log config:** no `log4j.properties` or `log4j.xml` discovered inside the deploy — relies on GeneXus runtime defaults baked into `GeneXus.jar` and Tomcat's `logs/` folder.
- **JDBC trace logging:** disabled — `client.cfg:94 JDBCLogEnabled=0`, `JDBCLogLevel=0`.
- **Application management endpoint:** disabled — `client.cfg:77 ENABLE_MANAGEMENT=0`.
- **Error tracking (Sentry / Rollbar / Bugsnag):** none configured.
- **APM (NewRelic / Datadog / Elastic APM):** none configured.

## Internationalization

- **Active language:** Portuguese (`client.cfg:57 LANGUAGE=por`, `LANG_NAME=Portuguese`, `culture=pt-BR`).
- **Date format:** `DMY` (`client.cfg:38`), 24-hour time (`client.cfg:41 TIME_FMT=24`), decimal separator `,` (`client.cfg:37`), thousand separator `.` (`client.cfg:86`).
- **Calendar JS locales available:** `web/calendar-pt.js` (active), plus en/es/de/it/ja/zh/ct.

## CI/CD & Deployment

- **CI:** none. No `.github/`, `.gitlab-ci.yml`, `Jenkinsfile`, `azure-pipelines.yml`, `bitbucket-pipelines.yml`, or build pipeline descriptors anywhere in the tree. Project is not even under git.
- **Deployment process (inferred):**
  1. Edit KB in GeneXus 12 IDE (Windows, against `GX_KB_Receituario.mdf`).
  2. Build → "Java Model" generation produces `Receituario/JavaModel/web/`.
  3. `Receituario/JavaModel/web/CALLMAKE.BAT` invokes `GXJMake.exe` to compile sources to `.class`.
  4. The directory is copied (manually or via a GeneXus "Deploy" action) to `C:\Program Files\Apache Software Foundation\Tomcat 8.5\webapps\ReceituarioJavaEnvironment\`.
  5. Tomcat hot-reloads via `<Context reloadable="true">` in `META-INF/context.xml`.

## Environment Configuration

**Critical runtime knobs (all in `Receituario/JavaModel/web/client.cfg` and its Tomcat mirror):**

| Key | Value | Meaning |
|---|---|---|
| `JDBC_DRIVER` | `org.postgresql.Driver` | PostgreSQL driver class. |
| `DB_URL` | `jdbc:postgresql://127.0.0.1:5432/saude-mandaguari` | Runtime app DB. |
| `USER_ID` / `USER_PASSWORD` | encrypted blobs | DB credentials (decrypt to role `sysmar`). |
| `DBMS` | `postgresql` | Selected DBMS. |
| `Theme` | `SysmarTheme` | Active web theme. |
| `LANGUAGE` | `por` | Portuguese (Brazil). |
| `USE_ENCRYPTION` | `SESSION` | Session-level URL encryption. |
| `POOLSIZE_RW` | `10` | JDBC pool size. |
| `GX_BUILD_NUMBER` | `118219` | GeneXus generator build. |
| `VER_STAMP` | `20260519.191424` | Last regen timestamp. |
| `CACHE_INVALIDATION_TOKEN` | `20265201014253` | Forces browsers to bust theme/JS caches when the KB regenerates. |

**Secrets location:** `client.cfg` itself — both the source copy at `Receituario/JavaModel/web/client.cfg` and the Tomcat deploy at `Tomcat 8.5/webapps/ReceituarioJavaEnvironment/WEB-INF/classes/client.cfg`. The credentials are encrypted with a GeneXus master key, but anyone with both `client.cfg` and `GeneXus.jar` can decrypt them — treat the file as sensitive.

## Webhooks & Callbacks

**Incoming:** none configured. No public webhook receivers (no Stripe/PagSeguro/Mercado Pago handlers) in the generated code.

**Outgoing:** none configured (no `HttpClient` use in generated procedures, no message-queue producer, no Kafka/RabbitMQ/JMS clients in `WEB-INF/lib/`).

## Summary of Active External Surface

The runtime application has a remarkably small external surface for a system this size:

- **Inbound:** HTTP only (Tomcat 8.5 on the standard `/ReceituarioJavaEnvironment` context).
- **Outbound:** PostgreSQL JDBC (loopback `127.0.0.1:5432`) only.
- **No third-party APIs** (no Stripe/AWS/Google/Microsoft Graph/SendGrid/Twilio).
- **No message queue.**
- **No file/object storage beyond local disk** (`PublicTempStorage/`, `PrivateTempStorage/`, `LayoutMetadata/`).
- **No email or SMS** (`SMTP_HOST` is blank).
- **OAuth2 / GAM / LDAP wired but turned off** (all `*Dummy` implementations, `LDAP_LOGIN=0`, `EnableIntegratedSecurity=0`).
- **Lucene full-text search is embedded** (no remote search service).
- **Web Notifications WebSocket** is registered (`GXWebSocket.jar`) but no handlers — effectively dormant.
- **Smart Devices offline-sync service** is declared in `GeneXus.services` but no SD client code is emitted in this Java Model regen.
- **Legacy Java applet print** (`printingappletsigned.jar`) — present but unusable on modern browsers; falls back to server-rendered PDF.

---

*Integration audit: 2026-05-21*
