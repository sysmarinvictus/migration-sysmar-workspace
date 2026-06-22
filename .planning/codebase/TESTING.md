# Testing Patterns

**Analysis Date:** 2026-05-21

## TL;DR

**There is no automated test suite.** This is GeneXus-generated Java output. There is no `src/test/`, no `pom.xml`, no `build.gradle`, no JUnit, no TestNG, no `*Test.java`, no `*Spec.java`, no fixtures directory, no CI config under `Receituario/`. Testing happens (a) inside the **GeneXus IDE** via Build & Run / Live Editing, and (b) **manually** in a browser against a deployed Tomcat instance.

Take that as a hard fact for planning purposes — do not "add tests under `JavaModel/web`". Anything you add there is deleted on the next regen (see `CONVENTIONS.md` §4).

---

## Test Framework

**Runner:** None present.

**Searches that came back empty:**
- `find /mnt/s/projetos/Receituario -iname '*test*.java'` → no matches
- `find /mnt/s/projetos/Receituario -iname '*junit*'` → no matches
- `find /mnt/s/projetos/Receituario -iname '*testng*'` → no matches
- `find /mnt/s/projetos/Receituario -maxdepth 5 -name 'pom.xml' -o -name 'build.gradle*' -o -name 'build.xml'` → no matches

No Maven, no Gradle, no Ant. The Java tree is shipped already-compiled (`.class` files next to `.java` files) by the GeneXus generator + its own `GXJMake.exe` / `CALLMAKE.BAT`.

**Assertion library:** None.

**Run commands:** None.

---

## How Testing Actually Happens

### 1. Inside the GeneXus IDE (primary path)

The KB developer iterates inside the GeneXus IDE on Windows. Validation steps:

- **Specify** — the IDE generates the Java for changed KB objects into `Receituario/JavaModel/web/`.
- **Build** — `GXJMake.exe` (`Receituario/JavaModel/web/GXJMake.exe`) compiles the Java to `.class`. Stale `.class` files and `LastCallTree.info` get rewritten.
- **Run** — the IDE deploys the webapp into the local Tomcat (path layout is documented in `createwebapplication.bat`) and opens the developer menu in a browser.
- **Live Editing** — for UI-only changes, the IDE pushes JS/HTML deltas without redeploying.

Errors that surface here:
- KB-level validation (referential integrity of attributes/transactions/domains).
- Java compile errors from `GXJMake.exe`.
- Runtime errors visible in the Tomcat console / browser.

### 2. Manual smoke testing against deployed Tomcat (production-shaped check)

Once deployed, verification is done by clicking through the live app. Key URLs and entry points found in the tree:

#### Developer menu (lists every web object)

- `static/developermenu.html` — see `JavaModel/web/developermenu.html`. Title: "GeneXus Developer Menu". Lists "Web Objects" linking to every `/servlet/<name>` endpoint. This is the *de facto* manual-test index in dev.

#### Servlet URLs (per `@WebServlet` annotations)

Every web object exposes `/servlet/<lowercase-object-name>`. Pattern confirmed in:
- `sau_pac.java` → `@WebServlet(value="/servlet/sau_pac")` (Paciente transaction form)
- `hindex.java` → `@WebServlet(value="/servlet/hindex")` (`wSysSAU - Sistema de Gestão da Saúde Pública Municipal` — the app landing page)
- `hwwsau_fun.java` → `/servlet/hwwsau_fun` ("Consultar Funcionários" — employee work-with)
- `hpromptsau_bai.java` → `/servlet/hpromptsau_bai` ("Selecionar Bairro" prompt)
- `hmasterpagelogin.java` → `/servlet/hmasterpagelogin` (login master page)

#### Authentication-related entry points (manual login + auth flows)

- `hmasterpagelogin.java` / `hmasterpagelogin_impl.java` — login page master.
- `hmudasenhalogin.java` / `_impl` — change-password during login.
- `cadastrarsenha.java`, `cadastrasenha1.java` / `_impl` — password registration / reset.
- `hacerto_banco.java` / `_impl` — DB-fix / migration screen (developer-facing).
- `bloqueiausuario.java`, `desbloqueiausuario.java` — block / unblock user (admin operations).
- `acessamodulo.java` — module access entry.
- `hnotauthorized*.java` (multiple variants: `hnotauthorized`, `hnotauthorizeduser`, `hnotauthorizedprompt`, `hnotauthorizedsuporte`, `hnotauthorizedsystem`) — error redirect targets when permissions fail.
- `hsessaoexpirada.java` / `_impl` — session-expired landing page.
- `pisauthorized.java` — procedure that backs the authorization check (`/servlet/pisauthorized`).

These are the obvious manual probes for "is the deployment working?" and "is auth working?".

#### Validation procedures useful for smoke-testing input handling

- `psau_val_cnpjcpf.java` — CPF/CNPJ check digit validator (called from `sau_pac` etc.). Good first procedure to sanity-check via direct URL or via the patient form.
- `psau_val_cns.java` — Cartão Nacional de Saúde validation.
- `psau_val_pis.java` — PIS number validation.
- `psau_valcep.java` — postal code validation.
- `psau_ver_pac.java` — patient existence verification.

#### Audit / log evidence

- `sau_log.java` + `sau_log_bc.java` + `SdtSAU_LOG.java` — every audited operation writes a `SAU_LOG` row. Inspecting the `sau_log` table after a manual scenario is the canonical "did it actually do anything?" check. The `usu_login` column on log rows ties the action to the operating user (see `CONVENTIONS.md` for the audit-attribute convention).

### 3. Full-text search reindex

`GxFullTextSearchReindexer.java` is a callable maintenance hook exposed by GeneXus (Lucene-backed via the `lucene-*.jar` bundled in `createwebapplication.bat`). Useful as a post-deploy verification step, not as a test.

---

## Test File Organization

Not applicable — no test files exist in the tree.

**Where one might *think* to add tests, and why you must not:**
- `Receituario/JavaModel/web/` — overwritten on every regen.
- `Receituario/JavaModel/compile/` and `state/` — generator-managed.
- Anywhere under `Receituario/sysmar/MDL*`, `GXSPC*`, `DATA00*`, `GXLOCK`, `Locks` — these are KB / runtime-storage directories.

If automated testing is needed, it has to live **outside** `Receituario/` entirely — in a separate test harness (e.g. a sibling Maven project that drives HTTP requests against the deployed Tomcat). That harness does not exist today.

---

## Mocking

Not applicable. No mocking framework in use.

If you need to exercise individual GeneXus procedures (`p*.java`) outside the web container, the GeneXus runtime supports a `main(String[] args)` entry on Main-program objects (see the `main` method in `hindex.java`) which calls `Application.init(GXcfg.class)` and `executeCmdLine`. This is the closest thing to a "unit harness" available — it requires the full `gxclass*.jar` runtime and a configured DB connection (same as production).

---

## Fixtures and Factories

Not applicable. Test data is whatever happens to exist in the configured PostgreSQL database (DBMS confirmed by the `Main DBMS: PostgreSQL` line in every generated header).

Local development DB state lives under sibling directories like `Receituario/DATA001`, `DATA002`, `DATA003` — these are runtime data files, not fixtures, and they are not source-controlled in this tree.

---

## Coverage

**Requirements:** None.

There is no coverage tooling, no coverage gate, no reporting target.

---

## Test Types

| Type             | Status              | Notes |
|------------------|---------------------|-------|
| Unit             | Absent              | No JUnit/TestNG; no `*Test.java`. |
| Integration      | Absent              | No integration harness; integration is implicit via the deployed webapp. |
| End-to-end       | Manual              | Performed by clicking through the deployed Tomcat instance. |
| Static analysis  | Absent              | No SpotBugs / PMD / Checkstyle config in `JavaModel/web`. The GeneXus IDE provides its own KB-level validation. |
| Smoke            | Manual via `developermenu.html` and the URLs listed above. |

---

## Common Patterns

### Async testing

Not applicable.

### Error testing

Manual. The generated code sets the `AnyError` byte after each operation (see `CONVENTIONS.md` §5 — Error handling). To "test" an error path you reproduce the condition via the UI and inspect the resulting `hnotauthorized*` redirect, the on-screen message, or the `sau_log` row written by the failure.

### Regression validation after a regen

Recommended manual sequence after any KB change + regen:

1. Open `static/developermenu.html` in the deployed app — confirm the changed objects appear.
2. Hit `/servlet/hindex` — landing page loads (login or main menu, depending on auth state).
3. Log in via `/servlet/hmasterpagelogin`.
4. Exercise the changed transaction (`/servlet/sau_<x>`) and its work-with (`/servlet/hwwsau_<x>`).
5. Query the `sau_log` table to confirm the audit row was written with the expected `usu_login`.
6. If validation logic changed, hit the affected `p*` procedure indirectly via the form, or directly at `/servlet/p<name>` if it's exposed.

---

*Testing analysis: 2026-05-21 — generator version 15_0_7-118219, no automated test infrastructure detected.*
