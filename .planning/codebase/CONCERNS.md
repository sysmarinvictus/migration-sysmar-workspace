# Codebase Concerns

**Analysis Date:** 2026-05-21
**Scope:** GeneXus-generated Java webapp `Receituario` (`/mnt/s/projetos/Receituario/JavaModel/web`) and deployed artifact `/mnt/c/Program Files/Apache Software Foundation/Tomcat 8.5/webapps/ReceituarioJavaEnvironment`.
**Ground truth:** GeneXus regen output; SQL Server KB (`.mdf`) is the source of truth; project is not a git repo. Domain: Brazilian health/medical prescription system (LGPD-regulated PHI).

---

## CRITICAL

### C1 — Database credentials committed in plain config files

**What it is:** PostgreSQL credentials (USER_ID + USER_PASSWORD) for the `saude-mandaguari` database are stored in clear-text config files in both the GeneXus source tree and the deployed webapp. The values appear to be GeneXus-encoded (Base64-looking, padded with `==`), not encrypted with a secret only the runtime can decrypt — GeneXus uses a fixed/derivable obfuscation key. Treat them as effectively plaintext.

**Where:**
- `/mnt/s/projetos/Receituario/JavaModel/web/client.cfg` lines 107–108 (`USER_ID=qVxDieU5...==`, `USER_PASSWORD=dudhgg...=`) under `[default|DEFAULT]`
- `/mnt/c/Program Files/Apache Software Foundation/Tomcat 8.5/webapps/ReceituarioJavaEnvironment/WEB-INF/classes/client.cfg` lines 107–108 — same credentials, deployed copy
- DB target: `DB_URL=jdbc:postgresql://127.0.0.1:5432/saude-mandaguari` (line 114, both files)

**Why it matters:** This is patient/health data under LGPD Art. 11 ("dados sensíveis"). Anyone with read access to the filesystem — backup tapes, devops tooling, file shares, the regenerating GeneXus IDE workstation, or an attacker with a directory-traversal/LFI on the Tomcat box — obtains direct DB credentials. GeneXus's obfuscation is reversible by the runtime and well-known to GX practitioners.

**Recommended action:**
1. **Immediately:** Rotate the PostgreSQL user `saude-mandaguari` password. Restrict that user to the application host's IP at the `pg_hba.conf` level. Audit DB logs for unauthorized access.
2. **Short term:** Use Tomcat JNDI DataSource (`USE_JDBC_DATASOURCE=1`, `JDBC_DATASOURCE=jdbc/saudemandaguari`) and store credentials in `META-INF/context.xml` with file-level OS permissions, or in Tomcat's `conf/server.xml` with `${env:DB_PASSWORD}` placeholders sourced from environment variables / a vault.
3. **GeneXus-side:** Set the connection at the KB level to use an external DataSource so regen does not re-emit credentials. Otherwise this finding will reappear after every regenerate.
4. **Cleanup:** When rotated, scrub the old credentials from any backups of `client.cfg`.

**Masked values shown in this report:** `USER_ID=qVxD…YkUk==`, `USER_PASSWORD=dudh…K5w=` (do not store the full values in this doc or in version control).

---

### C2 — Tomcat 8.5 is End-of-Life (since 2024-03-31)

**What it is:** The application is deployed on Apache Tomcat 8.5, which reached End-of-Life on **2024-03-31**. No further security patches are released. The server has been unsupported for ~14 months as of this analysis date (2026-05-21).

**Where:**
- Deploy path: `/mnt/c/Program Files/Apache Software Foundation/Tomcat 8.5/`
- `web.xml` declares `version="3.0"` (Servlet 3.0) — modest version, compatible with both Tomcat 9 and (via Jakarta migration) 10+
- `context.xml` uses `<JarScanFilter>` syntax — valid on Tomcat 8.5/9; would need verification on 10.1

**Why it matters:**
- Known unpatched CVEs accumulate on EOL Tomcat lines. This server fronts an LGPD-regulated PHI system.
- The deployed app uses `javax.servlet.*` (Servlet 3.0 namespace via `http://java.sun.com/xml/ns/javaee`), so it can move to Tomcat 9.0.x (still receives `javax.*` updates) without code changes. A move to Tomcat 10.1.x requires `jakarta.*` namespace migration — GeneXus must support that target in the generator.

**Recommended action:**
1. **Immediate path:** Upgrade to Tomcat **9.0.x** (latest patch). Same servlet namespace, drop-in for this webapp. Validates `context.xml`, `web.xml`, all dependency JARs.
2. **Verify GeneXus 15_0_7-118219 (build 118219, shown in `GXcfg.java`) supports Tomcat 9** — should be fine; GX targets Servlet 3.x. Confirm via GeneXus KB properties.
3. **Strategic path (12–18 months):** Investigate whether the GeneXus version in use can emit `jakarta.*` artifacts for Tomcat 10.1. If not, plan a GeneXus upgrade alongside Tomcat 10.x — GeneXus 17+ generally targets Jakarta EE 9+.
4. Until upgraded: front Tomcat with a reverse proxy (nginx/Apache) terminating TLS and applying WAF rules, and confirm Tomcat is bound to localhost only.

---

### C3 — `PublicTempStorage` configured as blob storage path; LGPD leak vector

**What it is:** `client.cfg` line 60 sets `CS_BLOB_PATH=PublicTempStorage`. In GeneXus, `CS_BLOB_PATH` is the directory used for user-uploaded files and generated/exported documents (PDFs, attachments). The literal directory `PublicTempStorage` exists in the deployed webapp root and is, by GeneXus convention, served as **publicly accessible static content** — anyone who knows or guesses a filename can fetch it without authentication.

**Where:**
- `/mnt/c/Program Files/Apache Software Foundation/Tomcat 8.5/webapps/ReceituarioJavaEnvironment/PublicTempStorage/` — exists (empty at scan time, but configured as the live blob path)
- `client.cfg:60` (`CS_BLOB_PATH=PublicTempStorage`)
- Contrast: line 51 `TMPMEDIA_DIR=PrivateTempStorage` correctly points temp media to a protected folder
- LGPD-relevant procedure exists: `buscadocumentolgpd.java` (Busca Documento LGPD) — explicit LGPD-related document lookup

**Why it matters:** If patient prescriptions/scans/PDFs traverse `CS_BLOB_PATH`, any path with a guessable or enumerable filename leaks sensitive health data under LGPD Art. 11. Tomcat by default serves the webapp root as static, so `https://host/ReceituarioJavaEnvironment/PublicTempStorage/<file>` is reachable directly. This is the canonical GeneXus PHI-leak vector.

**Recommended action:**
1. Audit what GeneXus objects actually write to `CS_BLOB_PATH` (search KB for `Blob`/`File`/`Save` attributes on `SAU_PAC`, `SAU_REM*`, `RECESP` transactions).
2. Move sensitive blobs to a private path: change `CS_BLOB_PATH=PrivateTempStorage` in KB-level properties; ensure no public link is generated to those files. Files needing user access must go through an authenticated GX procedure that validates session + ACL.
3. Add a Tomcat security-constraint or a reverse-proxy rule blocking direct access to `/PublicTempStorage/*` until proven safe.
4. Verify filenames are non-guessable (GUID-based, not sequential IDs or patient names).

---

## HIGH

### H1 — Hand edits to `JavaModel/web/*` will be wiped on next regen

**What it is:** The entire `JavaModel/web` tree is GeneXus-generated. The KB (SQL Server `.mdf`) is the source of truth. Every `.java` file in the root carries a header like:
```
File: HSAU_RECESP
Description: Receituário Controle Especial
Author: GeneXus Java Generator version 15_0_7-118219
Generated on: May 20, 2026 10:13:37.71
```
(observed in `/mnt/s/projetos/Receituario/JavaModel/web/hsau_recesp.java`)

All 468 `.java` files in the root carry this style of header (`grep -L "GeneXus Java Generator" ... | wc -l` = 0 missing). No obvious hand-edited file detected at scan time.

**Why it matters:** Any developer "quick fix" applied to a generated file (e.g., changing a SQL WHERE clause in a hsau_*.java, adding a log statement, tweaking validation) will be silently overwritten the next time GeneXus regenerates the model. This causes recurring bug-reintroduction and undermines patch hygiene.

**Recommended action:**
1. Treat `JavaModel/web/` as a read-only build output. All changes must go through the GeneXus KB (transactions, procedures, web panels, user controls).
2. Document the regen workflow for the team (who runs GeneXus, when, against which DB).
3. If a true "out-of-band" fix is ever needed in `web/`, immediately port it back to the KB and re-emit — never leave a divergence.
4. Recommend taking a `git` snapshot of `JavaModel/web/` after each regen so post-regen file diffs are visible — currently the project is not under git, which makes detecting accidental hand edits or unexpected regen drift very difficult.

---

### H2 — Project not under version control

**What it is:** Confirmed via the task context that the project is not a git repo. The deploy artifact and the regen output coexist on disk with no commit history.

**Why it matters:**
- No audit trail of what changed between regens — can't tell which patient-data-touching procedure changed when.
- No way to revert a bad regen without restoring from a filesystem backup.
- No code-review surface for KB changes that materialize as Java diffs.
- The KB itself (the SQL Server `.mdf`) is the authoritative state, but Java-side drift, manual config edits, or compromised drop-in JARs would go undetected.

**Recommended action:**
1. Put `JavaModel/web` under git, with a post-regen commit policy ("commit before deploy"). `.gitignore` the `*.class`, `*.002`, `*.VER`, `*.dll`, `*.exe`, `gxclassR.zip`, `gxclassD.zip`, `LastCallTree.info`, `bld12.info` — these are build outputs. **Do** commit `client.cfg`, `web.xml`, `context.xml`, the generated `.java` files (so regen diffs are visible). **Do not** commit `client.cfg` if you cannot first externalize the DB password (see C1).
2. Mirror/back-up the KB `.mdf` on a schedule separate from the Java tree. SQL Server backups are the only true "source of truth" recovery path.

---

### H3 — LGPD/PHI exposure surface needs review

**What it is:** This is a prescription system (`Receituario`) and the codebase confirms it stores LGPD Art. 11 "dados sensíveis" (health data): patient records (`SdtSAU_PAC` — "Paciente"), prescription transactions (`hsau_recesp*` — Receituário Controle Especial), medicine lots (`SAU_REMLOT`), and patient-info aggregates (`SdtSDT_INFO_PACIENTE_Item`, `SdtSDT_SAU_PESF_PAC`). A dedicated LGPD-related procedure exists (`buscadocumentolgpd.java`, "Busca Documento LGPD") which performs CPF/CNS masking (uses `XXX.` and `XXXXXXXX` patterns at lines 78, 92), suggesting some pseudonymization is implemented — but the surrounding access path was not validated.

**Where:**
- Patient transaction objects: `/mnt/s/projetos/Receituario/JavaModel/web/SdtSAU_PAC.java`, `SdtSDT_INFO_PACIENTE_Item.java`, `SdtSDT_SAU_PESF_PAC.java`
- Prescription transaction objects: `hsau_recesp.java`, `hsau_recespgeneral.java`, `hsau_recespsau_recesp1wc.java`, plus ~18 `*recesp*.java` files in `/mnt/s/projetos/Receituario/JavaModel/web/`
- External-patient existence probe: `existepacienteexterno.java` — a procedure that confirms whether a patient exists externally (potential PII enumeration endpoint if exposed without strong auth)
- LGPD document lookup: `buscadocumentolgpd.java`
- Audit log SDT: `SdtSAU_LOG.java` — suggests audit logging exists; verify it is actually written for every PHI access

**Why it matters:**
- LGPD requires (a) lawful basis for processing health data (Art. 11 §I/II), (b) data subject access rights with auditing, (c) breach notification within reasonable time, (d) DPO designation. A misconfigured endpoint, an unauthenticated procedure, or unscrubbed debug logging could cascade into a notifiable incident.
- `existepacienteexterno` is the kind of "does this CPF exist?" endpoint that is commonly weaponized for enumeration if it lacks rate limiting and authentication.
- Without a logging framework configured (see L1), it's unclear whether PHI access is being audited at all, or whether `SdtSAU_LOG` is actually populated by every read path.

**Recommended action:**
1. **Endpoint inventory:** Enumerate every web panel and procedure (`h*.java`, `a*.java`, `p*.java`) that returns patient data. Confirm each one is behind GeneXus's authentication (GAM or custom — check `web.xml` — note that GAM-related servlets in `web.xml` are wired to `*Dummy` classes, suggesting GAM is **not active** here; auth is happening elsewhere or is weak).
2. **Validate `existepacienteexterno`** authentication, rate limiting, and whether it returns true/false vs. richer data.
3. **PHI in logs:** Once a logging framework is configured (see L1), enforce a rule: no patient name, CPF, CNS, RG, diagnosis, or medication name in log messages. Configure a redaction filter.
4. **Audit trail:** Confirm `SAU_LOG` writes happen on every patient read; if not, consider a database-level audit table or trigger.
5. **DPO/legal:** Confirm with the Mandaguari municipality (DB name `saude-mandaguari` strongly suggests Mandaguari/PR Secretaria de Saúde) whether a DPO has been designated and a data-processing register (ROPA) exists.

---

### H4 — GAM (GeneXus Access Manager) servlets wired to `*Dummy` classes — auth model unclear

**What it is:** In `web.xml` (lines 47–65), the GAM OAuth-related servlets (`GAMOAuthSignIn`, `GAMOAuthCallback`, `GAMOAuthAccessToken`, `GAMOAuthUserInfo`, `GAMOAuthSignOut`) are all mapped to GeneXus *Dummy* implementation classes (`com.genexus.webpanels.agamextauthinputDummy`, `agamoauth20getaccesstokenDummy`, etc.). Same pattern for `GXOAuthAccessToken/Logout/UserInfo` → `*Dummy` (lines 24–36).

**Where:**
- `/mnt/c/Program Files/Apache Software Foundation/Tomcat 8.5/webapps/ReceituarioJavaEnvironment/WEB-INF/web.xml` lines 24–122

**Why it matters:** "Dummy" servlets are no-op stubs — they accept the request but do not perform real OAuth/GAM operations. This typically means GAM is not the active authentication mechanism, or it is wired in differently (e.g., session-based custom login through one of the GX procedures). Either way: **the authentication mechanism is not obvious from the deployment descriptor**, which makes security review hard and means the team must validate independently that PHI endpoints are actually protected.

`client.cfg:62` shows `USE_ENCRYPTION=SESSION` — encryption is session-scoped (URLs/params encrypted per-session). `EnableIntegratedSecurity=0` (line 78) — Integrated Security is off. So authentication is at the application/procedure level, not framework-level.

**Recommended action:**
1. Identify the actual login procedure (likely a `wp*login*.java` or similar in `JavaModel/web/`).
2. Verify session timeout, password storage hashing, lockout policy. The presence of `bloqueiausuario.java` (account lockout procedure) is a positive signal; confirm it is actually invoked on failed-login thresholds.
3. Confirm no PHI-returning endpoint can be reached without an active authenticated session, including the OAuth/GAM URL patterns — they currently return empty/dummy responses, but verify they are not accidentally usable as a bypass.

---

## MEDIUM

### M1 — Severely outdated dependency JARs in `WEB-INF/lib`

**What it is:** Direct inspection of `/mnt/c/Program Files/Apache Software Foundation/Tomcat 8.5/webapps/ReceituarioJavaEnvironment/WEB-INF/lib/` reveals the following versions stamped in filenames. Many are years past EOL.

| JAR (filename) | Concern |
|---|---|
| `log4j-1.2.15.jar` | **Log4j 1.x EOL since 2015-08-05.** Known CVEs: CVE-2019-17571 (deserialization RCE in `SocketServer`), CVE-2022-23305 (SQL injection in `JDBCAppender`), CVE-2022-23307 (Chainsaw deserialization), CVE-2022-23302 (`JMSSink` deserialization). Even if Chainsaw/SocketServer aren't used, the JAR's mere presence is flagged by every SCA scanner. |
| `commons-fileupload-1.3.2.jar` | CVE-2016-1000031 (deserialization → RCE); 1.5 is the patched line. **High risk for a webapp that accepts uploads** (and this one does — `GXObjectUploadServices` is mapped to `/gxobject`). |
| `commons-io-2.2.jar` | 2.2 is from 2012; current is 2.16+. Multiple CVEs, e.g. CVE-2024-47554. |
| `commons-lang-2.4.jar` | Commons Lang 2.x is EOL; should be Commons Lang3. |
| `commons-net-3.3.jar` | CVE-2021-37533 (FTP MITM). |
| `jackson-*-2.7.3.jar` (databind, core, annotations, jaxrs-*) | **Jackson 2.7.3 is from 2016.** Pre-2.10 jackson-databind has **dozens of polymorphic-deserialization RCE CVEs** (CVE-2017-7525 onward). Critical if any JSON endpoint deserializes untrusted input with default typing enabled. |
| `dom4j-1.6.1.jar` | CVE-2020-10683 (XXE-related deserialization). |
| `xercesImpl.jar`, `xml-apis-1.4.01.jar` | XML parser; risk depends on usage — XXE attack surface in any XML-parsing endpoint. |
| `bcprov-jdk15on-147.jar`, `bcpkix-jdk15on-147.jar` | BouncyCastle 1.47 from 2012. Multiple CVEs since. |
| `jersey-*-2.22.2.jar` | Jersey 2.22 from 2015 — EOL line. |
| `iText.jar` (no version in filename) | Likely iText 2.x (the last freely-licensed line before AGPL); pre-5.5.x has signature-bypass and other CVEs. Used for prescription printing — security relevance. |
| `poi*.jar` (Apache POI, no version) | Older POI versions have CVEs (XXE, deserialization). Need version verification. |
| `lucene-*-2.2.0.jar`, `lucene-highlighter-2.2.0.jar`, `lucene-spellchecker-2.2.0.jar` | Lucene 2.2.0 from ~2007 — wildly outdated. Used by `GxFullTextSearchReindexer`. |
| `mysql-connector-java-5.1.11-bin.jar` | MySQL Connector/J 5.1.11 from 2010. Numerous CVEs in 5.1.x line (e.g. CVE-2017-3523, CVE-2019-2692). Note: app uses **PostgreSQL** (per `client.cfg:113`), so this JAR may be unused dead weight — but its mere presence on the classpath is a deserialization concern. |
| `postgresql-9.1-902.jdbc3.jar` | PostgreSQL JDBC 9.1-902 (2014). The DB protocol is fine, but the driver has known CVEs (e.g. CVE-2018-10936). |
| `jtds-1.2.jar` | jTDS 1.2 (SQL Server/Sybase JDBC) — 1.3.x is the last release; both are abandoned. |
| `ojdbc6.jar` | Oracle JDBC 6 — pre-Java 7 build. Probably dead weight (PostgreSQL is the active DB) but on the classpath. |
| `validation-api-1.1.0.Final.jar` | OK in version, but pinned to Bean Validation 1.1 (2013). |
| `joda-time-2.8.2.jar` | Joda-Time 2.8.2 from 2015; current 2.12+. Lower risk (no major security CVEs). |

**Why it matters:** Jackson 2.7.3, Commons FileUpload 1.3.2, and Log4j 1.2.15 are three of the most exploited classes of Java deserialization/RCE vulnerabilities. The webapp accepts file uploads (`/gxobject`) and serves JSON via Jersey — both attack surfaces.

**Recommended action:**
1. **Quick wins (1–2 weeks):**
   - Replace `log4j-1.2.15.jar` with `reload4j` 1.2.25 (drop-in API-compatible, security-patched) OR migrate to `log4j2` (preferred). Don't keep Log4j 1.x.
   - Upgrade `commons-fileupload` to 1.5 (drop-in).
   - Upgrade Jackson 2.7.3 → 2.17+ (test API compatibility — there are some breaking changes around `ObjectMapper` defaults).
   - Remove unused drivers: `mysql-connector-*.jar`, `ojdbc6.jar`, `jtds-1.2.jar`, `sqlitejdbc-v056.jar`, `jt400.jar` (AS/400) — verify with the GeneXus KB which DBMS targets are actually used.
2. **GeneXus-side:** Most of these JARs are bundled by the GeneXus generator. Confirm whether the GeneXus version in use (build 118219) supports more recent dependencies; if it pins old versions, raise with GeneXus support or upgrade the generator.
3. **SCA tooling:** Run OWASP Dependency-Check or Snyk against `WEB-INF/lib/` for an authoritative CVE list.

---

### M2 — Brazilian regulatory: Receituário Especial (`recesp`) compliance not verified

**What it is:** The codebase implements Receituário Controle Especial (`hsau_recesp.java` Description: "Receituário Controle Especial", plus ~18 related `*recesp*.java` files in `JavaModel/web/`). In Brazil, controlled-substance prescriptions are subject to:
- **Portaria SVS/MS 344/1998** (Anvisa): list of controlled substances; mandates Receituário de Controle Especial (white, 2 vias, 30-day validity), Notificação de Receita A (yellow, narcotics), Notificação de Receita B (blue, psychotropics) and B2, Notificação de Receita Especial (retinoids, thalidomide).
- Required fields per prescription type: prescriber identification (name, CRM/CRO/CRF + UF), patient identification, address, quantity in numbers + extenso, signature, date.
- **Retention requirements:** the pharmacy retains the 1ª via; varying retention periods (commonly 2–5 years) depending on substance class.
- **Anvisa SNGPC** electronic reporting requirements for pharmacies handling controlled substances.

**Where:**
- `/mnt/s/projetos/Receituario/JavaModel/web/hsau_recesp.java`, `hsau_recespgeneral.java`, `hsau_recespsau_recesp1wc.java` and ~15 other `*recesp*.java` files
- Related procedures referenced in `LastCallTree.info`: `PSAU_PARPagIntRec`, `RSAU_RECESP_PAGINT`, `RSAU_RECESP`, `PSAU_VER_PAC`, `PSAU_VER_CNS`, `PSAU_VER_CNPJCPF`

**Why it matters:** Non-compliant electronic prescription handling (missing required fields, incorrect retention, no two-via printout, no traceability) can expose the issuing health unit to Anvisa sanctions and undermine the legal validity of issued prescriptions. The presence of `recesp` code does not, by itself, confirm regulatory compliance.

**Recommended action (validate with stakeholders, do not assume):**
1. Confirm with a regulatory/clinical lead (the Mandaguari Secretaria de Saúde or the responsible CRF pharmacist) that the issuing flow honours Portaria 344/98 requirements: which receita types are supported (A/B/B2/C/Especial), what is printed where, and what is retained.
2. Confirm whether the system integrates with or feeds **SNGPC**/Anvisa (likely a pharmacy concern, but verify boundaries).
3. Verify retention: the DB schema (KB-side, not visible here) must retain prescription records for the legally required period.
4. Validate signature: is a physical wet signature mandatory, or is an ICP-Brasil digital signature integrated? Currently no signing-cert handling is visible — the only crypto JARs are `bcprov`/`bcpkix` 1.47.
5. Verify CRM/CRO/CRF + UF capture on the prescriber profile (`SYS_PES`-related SDTs are visible).

---

### M3 — Tomcat `JarScanner` aggressively skipped — masks classpath misconfig

**What it is:** `/mnt/s/projetos/Receituario/JavaModel/web/context.xml` configures:
```xml
<JarScanner><JarScanFilter pluggabilitySkip="*" tldSkip="*" pluggabilityScan="GXWebSocket.jar"/></JarScanner>
```
This skips Servlet 3.0 pluggability scanning and JSP TLD scanning for **all** JARs except `GXWebSocket.jar`.

**Why it matters:**
- Pros: faster startup.
- Cons: if an upgraded JAR later relies on `ServletContainerInitializer` or `web-fragment.xml` (common for modern frameworks), it will silently not activate. Hidden footgun during dependency upgrades (see M1).

**Recommended action:** Acceptable for now (GX-emitted; speeds startup). Re-evaluate any time a non-GeneXus JAR is added or a major Servlet API change is in flight.

---

## LOW

### L1 — No logging configuration; observability black box

**What it is:**
- `log4j-1.2.15.jar` is on the classpath, but no `log4j.properties` or `log4j.xml` was found in the source tree or in `WEB-INF/classes/`. No `logback.xml` either. No SLF4J binding configuration.
- No application-emitted log file path is configured (`JDBC_LOG=`, `JDBCLogPath=`, `JDBCLogEnabled=0` in `client.cfg`).
- Tomcat's `catalina.out` and `localhost_access_log.*` will capture stdout/stderr and access logs respectively, but application logs effectively go to wherever the GeneXus runtime decides — typically stdout via the default Log4j configurator, which on Tomcat lands in `catalina.out`.
- No structured/JSON logging.
- No centralized log shipping (ELK/Graylog/Datadog).

**Why it matters:** Without app-level logging configuration, you can't reliably reconstruct a security incident, an LGPD access audit, or a prescription-issuance dispute. `SdtSAU_LOG.java` suggests a DB-side audit table, but verify it is actually populated on PHI reads (not just writes).

**Recommended action:**
1. Add a `log4j.properties` (or migrate to Log4j2 / Logback — preferred when log4j 1.x is replaced per M1) under `WEB-INF/classes/` with:
   - File appender to `${catalina.base}/logs/receituario.log` with rolling policy (e.g., daily, 30 days retention).
   - Separate audit appender for PHI-access logs (immutable/append-only file or syslog).
2. Configure log shipping to a central store with retention sufficient for LGPD audit trail requirements.
3. Make sure no log statement emits patient identifiers (CPF, CNS, full name, diagnoses, prescribed substances) — emit pseudonymized IDs instead.

---

### L2 — i18n: single-locale Portuguese deployment

**What it is:**
- `client.cfg` lines 57–58: `LANGUAGE=por`, `LANG_NAME=Portuguese`; lines 82–88 define the `Portuguese` block (`culture= pt-BR`, `decimal_point=,`, `date_fmt=DMY`, `time_fmt=24`).
- One messages file: `/mnt/s/projetos/Receituario/JavaModel/web/messages.por.txt`. No `messages.eng.txt` or `messages.esn.txt`.
- One resources subdir: `/mnt/s/projetos/Receituario/JavaModel/web/Resources/Portuguese/`.
- File comment headers contain mojibake (e.g., `Description: Receitu�rio Controle Especial` — broken `á`), indicating the GeneXus toolchain or storage layer is not handling UTF-8 consistently across the build pipeline.

**Why it matters:**
- Single-locale is acceptable for a Brazilian municipal health system; not a defect.
- The mojibake in source comments is cosmetic but suggests encoding mismatch (probably CP1252 vs UTF-8). The XML files declare `UTF-8`, the messages file is `.txt` without a BOM — confirm encoding before running searches/builds on Linux hosts where charset defaults differ.
- The deployed app is Pt-BR-only; document this as an explicit product constraint rather than an oversight.

**Recommended action:**
1. Standardize source-file encoding (UTF-8) at the GeneXus KB level if possible.
2. Document Pt-BR as the supported locale.
3. Accessibility (a11y) was not in scope of this scan but should be checked separately if the system is consumed by health professionals with assistive tech: confirm WCAG 2.1 AA on the SysmarTheme.

---

### L3 — Developer Menu and dev artifacts in deployed webapp

**What it is:**
- `/mnt/s/projetos/Receituario/JavaModel/web/developermenu.html` and `/mnt/s/projetos/Receituario/JavaModel/web/devmenu/` (developermenu.js, css, QR-code assets for iOS/Android KBN installs) are present in the source tree. These are GeneXus's "Developer Menu" for browsing web objects and installing Smart Devices apps.
- Build artifacts that should not be in a production webapp: `LastCallTree.info`, `bld12.info`, `gxtemplate.json`, `gxclassR.zip`, `gxclassD.zip`, `Images.txt`, `gxCompressInfo.data`, `CALLMAKE.BAT`, `GXJMake.exe`, `GXScanner.jar`, `*.002`, `*.VER`, `GXDIB32.DLL`.

**Why it matters:**
- The Developer Menu lists every web object — if reachable in production it gives an attacker a free site map of every web panel and procedure URL.
- Other artifacts disclose build metadata (`bld12.info` reveals model paths, `LastCallTree.info` reveals procedure names).

**Recommended action:**
1. In the deployed webapp, restrict `/developermenu.html`, `/devmenu/*`, and `/gxmetadata/*` via a `<security-constraint>` in `web.xml` or a reverse-proxy rule (deny in production, allow only from internal admin IPs).
2. Exclude `.bat`/`.exe`/`.dll`/`.002`/`.VER`/`.info`/`Make*` artifacts from the deploy WAR — those are dev/regen tooling, not runtime. Confirm what the GeneXus deploy step copies and trim it.
3. Confirm `gxmetadata/` exposure on the deployed instance — KB metadata reachable via HTTP is a recon goldmine.

---

### L4 — Java source files shipped alongside `.class` files

**What it is:** Both `.java` and corresponding `.class` files are present in the source tree (e.g., `hsau_recesp.java` + `hsau_recesp.class`). The deployed `WEB-INF/classes` directory contains only `.class` files (verified) — good. But the GeneXus source tree at `/mnt/s/projetos/Receituario/JavaModel/web/` mixes both.

**Why it matters:** Source files in the deploy tree (if mistakenly copied) leak business logic. The current deployed `WEB-INF/classes` looks clean, but verify after every redeploy that no `.java` files are copied.

**Recommended action:** Confirm the deploy step (whatever produces `ReceituarioJavaEnvironment/`) only includes `.class` files and required resources, never `.java` sources.

---

## Summary table

| ID | Severity | One-line | First action |
|----|----------|----------|-------------|
| C1 | Critical | DB credentials in `client.cfg` (both source and deployed) | Rotate password; move to JNDI DataSource |
| C2 | Critical | Tomcat 8.5 EOL (since 2024-03-31) | Upgrade to Tomcat 9.0.x (drop-in) |
| C3 | Critical | `PublicTempStorage` is the CS_BLOB_PATH — public PHI leak vector | Move blobs to `PrivateTempStorage`; block path at proxy |
| H1 | High | Regen wipes any hand edits to `JavaModel/web` | Treat as build output; enforce KB-only changes |
| H2 | High | Project not under git | Initialize git; commit post-regen |
| H3 | High | LGPD/PHI exposure surface unaudited | Endpoint inventory; auth & log audit |
| H4 | High | GAM servlets are Dummy stubs — auth model unclear | Identify and validate the real auth path |
| M1 | Medium | Severely outdated JARs (Log4j 1.x, Jackson 2.7, FileUpload 1.3.2, …) | Upgrade Log4j → reload4j/Log4j2; upgrade Jackson; upgrade FileUpload |
| M2 | Medium | Receituário Especial (Portaria 344/98) compliance not verified | Validate with regulatory lead/CRF |
| M3 | Medium | `JarScanner` skips pluggability for non-GX JARs | Acceptable; re-eval on upgrades |
| L1 | Low | No log4j/SLF4J config; observability black box | Add log4j.properties with audit appender |
| L2 | Low | Single-locale Pt-BR; encoding mojibake in comments | Document; standardize UTF-8 |
| L3 | Low | Developer Menu and dev artifacts in deploy tree | Restrict /developermenu, /gxmetadata, /devmenu |
| L4 | Low | `.java` sources shipped alongside `.class` in source tree | Verify deploy step strips sources |

---

*Concerns audit: 2026-05-21*
