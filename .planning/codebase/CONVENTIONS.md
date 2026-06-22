# Coding Conventions

**Analysis Date:** 2026-05-21

> **Critical context — read first.** This codebase under `Receituario/JavaModel/web` is **GeneXus regeneration output** (GeneXus Java Generator `15_0_7-118219`, generated 2026-05-20). Almost everything in this document is *imposed by the generator*, not chosen by a human. The Knowledge Base (KB) — managed inside the GeneXus IDE — is the source of truth. Conventions listed here are **observation of the generator's output**, useful for reading and navigating the code, but **not edit targets**.
>
> See the "No-Hand-Edits Rule" section below — it is non-negotiable.

---

## 1. Domain Naming Glossary (authoritative)

Tables and columns in this product are globally unique repo-wide and capped at 30 chars. A name appearing in one table refers to the same concept in any other table. The first three letters of a domain entity are always one of the prefixes below. Memorize these — they make hundreds of file/class names instantly readable.

### Entity prefixes

| Prefix   | Portuguese                | English                   |
|----------|---------------------------|---------------------------|
| `fun`    | funcionario               | employee                  |
| `pes`    | pessoa                    | person                    |
| `pac`    | paciente                  | patient                   |
| `pro`    | profissional de saude     | health professional       |
| `rem`    | remedio                   | medicine / drug           |
| `esp`    | especialidade             | medical specialty         |
| `uni`    | unidade de atendimento    | service / care unit       |
| `vac`    | vacina                    | vaccine                   |
| `usu`    | usuario do sistema        | system user               |
| `recesp` | receituário especial      | special prescription      |

### `sau_*` (saúde) reference tables

| Table       | Meaning                            |
|-------------|------------------------------------|
| `sau_bai`   | bairro (neighborhood)              |
| `sau_cbor`  | Cadastro Brasileiro de Ocupações   |
| `sau_concla`| conselho de classe                 |

### Column conventions

- `usu_login` = the login of the user performing the operation. Used as the audit-trail attribute across tables (e.g. on `sau_log`, transaction rows, etc.).
- Column attributes in the generated Java appear as `A<n><Prefix><FieldName>` (e.g. `A700PacPesCPFCNPJ`, `A153PacPesMunCod`, `A3PacPesCod`). See `sau_pac_impl.java` and `sau_pac_bc.java` for canonical examples.

---

## 2. File-Naming Prefixes (GeneXus generator)

Every `.java` file under `JavaModel/web` is named after its KB object, lowercased, with a one-letter or short prefix encoding the object type. Counts below are from the live filesystem listing.

| Prefix       | Object kind                                    | Example                                | Count* |
|--------------|------------------------------------------------|----------------------------------------|--------|
| `sau_<x>`    | Transaction (table CRUD form)                  | `sau_pac.java`, `sau_fun.java`         | ~71    |
| `<name>_bc`  | Business Component (silent transaction handle) | `sau_pac_bc.java`, `sau_log_bc.java`   | 4      |
| `<name>_impl`| Implementation half of any web object          | `sau_pac_impl.java`, `hindex_impl.java`| paired with stub |
| `h<name>`    | Web Panel (page) — the "h" is the legacy "HTML object" tag in the KB | `hindex.java`, `hmasterpagelogin.java` | ~40+ |
| `hpromptsau_<x>` | Prompt (selector popup) for a `sau_*` table | `hpromptsau_bai.java`, `hpromptsau_pac2.java` | ~58 |
| `hwwsau_<x>` | Work With (list / search grid) for a `sau_*` table | `hwwsau_fun.java`, `hwwsau_pac.java`   | ~26 |
| `hviewsau_<x>` | View (detail page) for a `sau_*` table       | `hviewsau_fun.java`                    | ~22 |
| `hsau_<x>general` | "General" tab view of a table             | `hsau_fungeneral.java`, `hsau_pacgeneral.java` | ~42 |
| `p<name>`    | Procedure (callable routine, no UI)            | `psau_val_cnpjcpf.java`, `pisauthorized.java` | ~56 |
| `psau_<x>`   | Procedure tied to a `sau_*` domain operation   | `psau_inc_pac.java`, `psau_checarperm.java` | (subset of above) |
| `r<name>` / `rsau_<x>` | Report                               | `rsau_pacextres.java`, `rsau_recesp.java` | 8 |
| `rwd<name>`  | Responsive Web Design variant of a master/panel| `rwdmasterpage.java`, `rwdpromptmasterpage.java` | 6 |
| `gxdomain<x>`| Enumerated GeneXus domain (small lookup type)  | `gxdomainpage.java`, `gxdomaintrnmode.java` | 13 |
| `Sdt<NAME>`  | SDT (Structured Data Type) — DTO/record        | `SdtSAU_PAC.java`, `SdtContext.java`   | 9 |
| `StructSdt<NAME>` | Internal struct backing an SDT            | `StructSdtSAU_LOG.java`                | paired with `Sdt*` |
| `GX<Name>`, `Gx<Name>` | GeneXus runtime / config helper      | `GXcfg.java`, `GxWebStd.java`, `GxObjectCollection.java` | small set |

\*Counts are from `find … -maxdepth 1 -name '<pattern>.java'` against `Receituario/JavaModel/web`.

### Pairing convention

Almost every web object exists as a **pair**: a thin servlet stub and an `_impl` body.

- `<name>.java` — extends `GXWebObjectStub`, carries the `@javax.servlet.annotation.WebServlet(value="/servlet/<name>")` annotation, delegates `doExecute` / `init` to its `_impl`. See `hindex.java`, `sau_pac.java`, `hmasterpagelogin.java`.
- `<name>_impl.java` — extends `GXDataArea`, contains all logic (event handlers, SQL cursors, layout emission).

Procedures (`p*`) extend `GXProcedure` directly and do not have an `_impl` stub split.

Business Components (`<name>_bc.java`) extend `GXWebPanel` and implement `IGxSilentTrn` — they are the silent (no-UI) version of a transaction used from procedures and other panels (`sau_pac_bc.java`).

---

## 3. Language Conventions

**Identifiers and runtime code:** Portuguese (Brazilian — PT-BR). Examples from filenames:
`buscaloginusuario`, `bloqueiausuario`, `cadastrasenha1`, `clonarcontroleacesso`, `acessamodulo`, `permitircadastrosemcpf`, `interacaomedicamentosa`.

**File-header `Description:` lines and `getServletInfo()` strings:** Portuguese (`Paciente`, `Selecionar Bairro`, `Consultar Funcionários`, `Informações do Funcionário`, `wSysSAU - Sistema de Gestão da Saúde Pública Municipal`). Note: these strings often appear with mojibake (`Funcion�rios`) in the generated `.java` because of the encoding the generator emits.

**GeneXus runtime / framework identifiers:** Mixed Portuguese/Spanish/English following GeneXus product norms — `gxfirstwebparm`, `getInsDefault`, `confirm_090`, `checkExtendedTable093`, `beforeValidate`, `AnyError`, `Gx_mode` (with values `"INS"`, `"UPD"`, `"DLT"`). Don't try to translate these; they're framework primitives.

**JavaScript runtime config:** see `gxcfg.js` — `gx.setLanguageCode("por")`, `gx.setDateFormat("DMY")`, `gx.setDecimalPoint(",")`, `gx.setThousandSeparator(".")`. The shipped target locale is Brazilian Portuguese.

**Static UI assets (JS framework):** English (`gxgral.src.js`, `developermenu.html` — the dev menu is English: "Browse Web Objects", "Install iOS Apps", etc.).

---

## 4. The No-Hand-Edits Rule

**Hand edits to anything under `Receituario/JavaModel/web` are lost on the next GeneXus regen.**

Concretely:
- Every `.java` file in this tree begins with a comment block of the form:
  ```
  /*
                 File: SAU_PAC
          Description: Paciente
               Author: GeneXus Java Generator version 15_0_7-118219
         Generated on: May 20, 2026 10:12:59.89
         Program type: Callable routine
            Main DBMS: PostgreSQL
  */
  ```
  Any file carrying this banner is generator output. Touching it is a no-op (the next "Build" or "Run" from the GeneXus IDE overwrites it).
- The corresponding `.class` files sit alongside the `.java` files (e.g. `sau_pac.class`, `sau_pac__default.class`), confirming this is a build output directory, not a source directory.
- `.VER` files (`GJS118219.VER`, `GxMask.1.0.2.VER`, `HMask.0.47.VER`) and `.002` files (`GXHPRO12.002`, `GXPPRO12.002`, etc.) are generator-version markers and tooling binaries — never edit.

**Where real changes go:**
1. Open the GeneXus IDE.
2. Modify the relevant KB object: Transaction, Web Panel, Procedure, Domain, Theme, etc.
3. Save → Build → Run. The IDE regenerates the affected `.java`/`.class`/`.js` files in this directory.
4. Verify in the running app (Tomcat / Live Editing — see `TESTING.md`).

If you find yourself wanting to add a helper class, a unit test, or "just a quick fix" inside `JavaModel/web`, **stop**. The right place is the KB.

The narrow exceptions where files in this tree are not regenerator output are infrastructure files like `web.xml`, the `web*.xml` variants, `context.xml`, `contextScanFilter.xml`, `CloudServices.config`, `GeneXus.services`, and `createwebapplication.bat`. These are still produced by the GeneXus tooling chain (or copied from a template), and should be configured via KB / environment properties rather than directly edited.

---

## 5. Code Style (observed in generated output)

Treat the items below as **read-only reading aids**, not style guidance you can enforce.

### Formatting

- 3-space indent inside class bodies, 6-space for nested blocks.
- Opening brace on its own line ("Allman" style), even for one-line methods.
- Trailing spaces around parentheses and operators are common (`public sau_pac_bc( int remoteHandle )`, `if ( AnyError == 0 )`). This is generator style.
- `final` is added to almost every top-level class (`public final class sau_pac extends GXWebObjectStub`).

### Imports

Every generated file uses **default-package** classes (no `package` statement) and the same five star-imports, in this order:

```java
import com.genexus.*;
import com.genexus.db.*;
import com.genexus.webpanels.*;   // omitted in procedure files
import java.sql.*;
import com.genexus.search.*;
```

Procedure files (`p*.java`) reorder slightly (`java.sql.*` first, then `com.genexus.db.*`, then `com.genexus.*`) — see `psau_val_cnpjcpf.java`. This ordering is generator-controlled, not a convention to follow.

### Identifier conventions in generated code

- Class names: all lowercase with underscores (`sau_pac`, `hpromptsau_bai`, `psau_val_cnpjcpf`). They match the file name exactly.
- "Stub vs impl" split: `<name>` is the servlet, `<name>_impl` is the panel logic.
- Variables prefixed `A<n>` are attributes (columns), `AV<n>` are local variables, `Z<n>` are "before update" snapshots, `n<n>` are null-flag booleans (`n153PacPesMunCod`). All emitted by the generator.
- `Gx_mode` holds the transaction mode: `"INS"`, `"UPD"`, `"DLT"`.
- `AnyError` is the global error-state byte checked after every operation.

### Comments

Generated code carries the header banner described in §4 and very sparse inline comments (`/* GeneXus formulas */`, `/* Output device settings */`, `/* Execute user event: After Trn */`). Inline comments come from the KB event names; they reflect KB structure rather than free-form documentation.

### Error handling

Procedural style — after every DB / cross-cutting call, check the `AnyError` byte:

```java
if ( AnyError == 0 )
{
   // continue
}
```

No exceptions are thrown from generated code. The runtime (`GXProcedure`, `GXDataArea`) handles I/O exceptions internally and sets `AnyError` / `GxWebError`.

---

## 6. JavaScript / Front-end Conventions

Each web object emits a companion `<name>.js` next to its `.java` (e.g. `hindex.js`, `cadastrasenha1.js`, `hacerto_banco.js`). These are minified-ish, KB-driven, and follow the same no-hand-edits rule.

The shared framework JS lives in:
- `gxgral.js` / `gxgral.src.js` — GeneXus general client runtime
- `gxcfg.js` — per-build configuration (language, date format, build number)
- `gxdec.js`, `gxprint.js`, `gxtimezone.js`
- `calendar*.js` — date picker (one file per UI locale)
- `autosuggest.css`, `calendar-system.css`, `container.css` — base styles
- `bootstrap/`, `SlideDownMenu/` — third-party UI assets bundled by the generator
- `GxMask-master/`, `HMask/` — input-mask plugin and version stamp

---

## 7. Deployment Conventions

`createwebapplication.bat` documents the deployment shape (Tomcat webapp layout):

- WAR-style directory: `webapps/<appname>/WEB-INF/{classes,lib}`, `webapps/<appname>/{static,gxmetadata,Metadata,...}`.
- Servlets are registered both via `web.xml` (GeneXus runtime servlets — see `GXObjectUploadServices`, `GXResourceProvider`, `GXValidService`, `GAMOAuth*`) and per-class `@WebServlet(value="/servlet/<name>")` annotations on every `<name>.java`.
- URL pattern for every panel/procedure: `/servlet/<lowercase-object-name>` (e.g. `/servlet/sau_pac`, `/servlet/hindex`, `/servlet/hwwsau_fun`).
- A `developermenu.html` (static) is shipped at `static/developermenu.html` — entry point for browsing every generated web object during development.

---

## 8. What You Should Actually Document Yourself

The KB-side conventions (naming, event handlers, data providers, themes) live in the GeneXus IDE and are not visible from this tree. When adding new attributes or domains, the *real* convention reference is:

1. The KB itself, opened in the GeneXus IDE.
2. Internal team docs about the `sau_*` schema (if any exist outside the KB).
3. Existing KB objects of the same kind — copy their structure.

Do not invent conventions for the generated Java. Do invent / enforce conventions inside the KB.

---

*Convention analysis: 2026-05-21 — generator version 15_0_7-118219, regen timestamp 2026-05-20.*
