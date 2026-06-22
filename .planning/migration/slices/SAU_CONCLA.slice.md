# SLICE-SPEC — SAU_CONCLA (Conselho de Classe)

> Wave-1 warm-up leaf lookup. Professional **licensing board / class council** (e.g. CRM=medicine,
> COREN=nursing, CRF=pharmacy). Referenced by `SAU_PRO` (profissionais). Schema below is taken from
> the **GeneXus reorg DDL** (`sysmar/MDL002/GEN12/SAU_CONCLAConversion.xml`) — authoritative for
> types, but **live-DB introspection was NOT run** (no PostgreSQL reachable in this environment);
> `gx-schema-mapper` should confirm against `saude-mandaguari` before this advances past `specced`.

```yaml
slice: SAU_CONCLA
domain: conselho-classe
description: Conselho de Classe
wave: 1
complexity: S            # sau_concla_impl.java ~100 KB
primary_table: SAU_CONCLA
constellation:
  transaction: sau_concla.java
  impl: sau_concla_impl.java          # ~100 KB — rule source (2324 lines)
  bc: null                            # not a Business Component (no gxmetadata json)
  workwith: null                      # no hwwsau_concla in this regen — list via prompt/Work-With pattern
  view: null
  prompts: [hpromptsau_concla.java]   # FK lookup popup → /api/conselhos-classe/lookup
  sections: []
  sdt: null
depends_on: []                        # owns one table; no outbound FK to migrate first
# reverse reference (NOT a depends_on): SAU_PRO.ConClaCod → SAU_CONCLA.ConClaCod drives the delete
#   guard (R3). SAU_PRO is Wave 4 (un-migrated) → implement the guard as a native count query against
#   SAU_PRO until SAU_PRO is migrated, then replace with a JPA relationship/derived count.
# cross-cutting: legacy SAU_LOG audit is NOT written by this transaction (TableAccess lists only
#   SAU_PRO + SAU_CONCLA) — non-PHI reference data; no @Auditable required.
fields:
  - { gx: A238ConClaCod,  name: codigo, type: smallint, pk: true, nullable: false, picture: "ZZ9",
      note: "user-entered code 0..999 (Autonumber=No, Autogenerate=No). src sau_concla_impl.java:501-517, Conversion.xml AttriId 238" }
  - { gx: A179ConClaSigra, name: sigla, type: string, len: 10, nullable: true,
      note: "council acronym e.g. CRM/COREN/CRF. VARCHAR(10) AllowNulls=Yes. src Conversion.xml AttriId 179, edit sau_concla_impl.java:316" }
  - { gx: A861ConClaNom,  name: nome,  type: string, len: 100, nullable: true,
      note: "council full name. VARCHAR(100) AllowNulls=Yes (UI edit shows 80 cols). src Conversion.xml AttriId 861, edit sau_concla_impl.java:327" }
keys:
  pk: [A238ConClaCod]
  unique: [{ index: ISAU_CONCLA, cols: [A238ConClaCod] }]   # = the PK index
  fks: []                                                    # no outbound FK
ddl_baseline: |
  CREATE TABLE SAU_CONCLA (ConClaCod smallint NOT NULL, ConClaSigra VARCHAR(10),
    ConClaNom VARCHAR(100), PRIMARY KEY(ConClaCod));
  # already part of the existing schema — entity maps to it; no new Flyway DDL (ddl-auto=validate).
endpoints:
  - { method: GET,    path: /api/conselhos-classe,        from: list/search,  role: SAUDE_CADASTRO }
  - { method: GET,    path: /api/conselhos-classe/{id},   from: view }
  - { method: POST,   path: /api/conselhos-classe,        from: sau_concla INS, role: SAUDE_CADASTRO }
  - { method: PUT,    path: /api/conselhos-classe/{id},   from: sau_concla UPD, role: SAUDE_CADASTRO }
  - { method: DELETE, path: /api/conselhos-classe/{id},   from: sau_concla DLT, role: SAUDE_CADASTRO }
  - { method: GET,    path: /api/conselhos-classe/lookup, from: hpromptsau_concla, desc: autocomplete }
phi_fields: []            # professional-licensing reference data — no patient PHI
auth:
  roles_required: [SAUDE_CADASTRO]
  notes: >
    LEGACY HAS SECURITY DISABLED for this transaction — sau_concla.java:33 IntegratedSecurityEnabled()
    returns false (no per-permission check). The new app should still gate CRUD behind a role;
    SAUDE_CADASTRO proposed to match the SAU_ESP reference slice — confirm with the access model.
parity:
  legacy_base: "http://<host>/ReceituarioJavaEnvironment"
  scenarios:
    - list default
    - get by id (existing)
    - get by id (404)
    - insert valid (codigo + sigla + nome)
    - insert valid (codigo only — sigla/nome null, allowed)
    - insert duplicate codigo (reject — R2)
    - insert codigo out of range > 999 (reject — R1)
    - update sigla + nome
    - update attempting to change codigo (rejected / treated as new key — R2)
    - delete unused (ok)
    - delete referenced by a profissional in SAU_PRO (reject — R3)
status: verified       # parity 11/11 PARITY 2026-06-22. Report: backend/src/test/resources/parity/conselho-classe/PARITY-REPORT.md.
# Entity↔live types match. Client-supplied PK (R1 range, R2 dup→409); R3 delete-guard vs SAU_PRO→409.
# codigo>999 rejects via 400 (Bean Validation) not 422 — status nuance (RF6). No business divergences.
```

## Migration generation log (migrate-slice, 2026-06-10)
Backend, frontend, and test code generated in `receituario-modern/`, following the SAU_ESP reference:

- **Backend** `backend/.../conselhoclasse/`: `domain/ConselhoClasse` (entity, `Short` PK ↔ `smallint`),
  `dto/ConselhoClasseDtos`, `mapper/ConselhoClasseMapper`, `repository/ConselhoClasseRepository`,
  `service/ConselhoClasseService` (R1/R2/R3/R5), `web/ConselhoClasseController` (CRUD + lookup,
  `@PreAuthorize('SAUDE_CADASTRO')`).
- **Flyway** `V1__baseline.sql`: added `SAU_CONCLA` + a **minimal `SAU_PRO` stub** (ProCod, ConClaCod)
  for the R3 delete-guard. The full `SAU_PRO` (XL, Wave 4) will expand this stub.
- **Tests**: `ConselhoClasseServiceTest` (one per rule), `ConselhoClasseControllerIT`
  (per-endpoint + security + rules), `parity/ConselhoClasseParityIT` (disabled stubs).
- **Frontend** `frontend/src/features/conselho-classe/`: `schema` (Zod R1/R5), `api`, list page,
  form page, `schema.test`; routes + nav registered in `app/App.tsx`.

**BUILT & TESTED (2026-06-10)** — toolchain installed in-env (OpenJDK 21 + Maven via apt; Docker
engine via `apt docker.io`). Results:
- Backend unit (`ConselhoClasseServiceTest`): **6/6** pass.
- Backend integration (`ConselhoClasseControllerIT`, Testcontainers Postgres 16): **12/12** pass —
  full HTTP→service→JPA→PostgreSQL, incl. 401/403 security and rules R1/R2/R3/R5.
- Frontend (`schema.test.ts`): **4/4** pass; `tsc --noEmit` clean.
- `Short`↔`smallint` validated against the Flyway baseline DDL under `ddl-auto=validate` (no mismatch).

Fixes applied during build (committed in `receituario-modern`):
- Entity no-arg constructor `protected`→`public` (also fixed the same latent bug in the SAU_ESP
  reference) — services instantiate entities from a sibling package.
- `SecurityConfig`: added a 401 `authenticationEntryPoint` and permitted the `/error` dispatch so
  unauthenticated→401 and authenticated-but-forbidden→403 (the contract the IT encodes).

Environment notes for whoever runs this elsewhere:
- `AbstractIntegrationTest` now uses a **singleton Testcontainers Postgres** (one container + one
  cached Spring context shared by all IT classes). Both `*ControllerIT` run together → 24/24 pass.
- **`mvn verify` runs the full suite**: surefire runs unit `*Test`, `maven-failsafe-plugin` runs the
  Testcontainers `*IT` (parity excluded). `mvn -Pparity verify` runs only the `@Tag("parity")` ITs.
- testcontainers-bom bumped to **1.21.4** so docker-java negotiates with Docker 28+ (the 1.20.4
  client spoke API 1.32, which modern Docker rejects). No `-Dapi.version` flag needed.

Remaining open question (live DB): confirm `Short`↔`smallint` matches the **production** column; if
it's actually `integer`, switch the entity/DTO/PK type to `Integer` and the baseline to `INTEGER`.

## Parity — DEFERRED (2026-06-12)
`/verify-parity` could not run: this environment has no reachable GeneXus instance and no
non-production DB. Investigation found the legacy app can't be stood up from the repo alone —
only 2 of 58 `Conversion.xml` files carry full DDL (the rest are ALTER-only; the complete schema
lives in the live `saude-mandaguari` DB), there's no data snapshot, the `com.genexus` runtime ships
as an opaque `gxclassR.zip`, Tomcat 9/javax isn't in apt, and the `client.cfg` DB creds are
GeneXus-encrypted. `ConselhoClasseParityIT` stubs remain ready. **To run parity later, provide:**
a reachable legacy base URL (run in the existing GeneXus/Tomcat env) + a **non-production** DB copy
(host/db/read-user) for result-set and resulting-row diffing. Status stays `tested` until then.

## Mined rules (from sau_concla_impl.java — confidence noted; cite line refs)

- **R1** — `codigo` is the primary key, a user-entered `smallint` in range **0–999** (`picture ZZ9`).
  Out-of-range input is rejected with `GXM_badnum`. The code is **not** auto-numbered/auto-generated.
  `kind: validation` · `confidence: high` · src `sau_concla_impl.java:501-517`, `Conversion.xml`
  (Autonumber=No, Autogenerate=No, AllowNulls=No) · `target: ConselhoClasseServiceTest#rejectsCodigoOutOfRange`,
  `#requiresCodigo`.
- **R2** — On INSERT `codigo` must be unique; a duplicate is rejected (`DuplicatePrimaryKey`,
  `GXM_noupdate`). On UPDATE the key is immutable (changing it is treated as a new-key insert / blocked).
  `kind: validation` · `confidence: high` · src `:1238-1244` (insert), `:964-973` (key change) ·
  `target: #rejectsDuplicateCodigo`, `#codigoImmutableOnUpdate`.
- **R3** — **DELETE is blocked when the council is referenced by a Profissional** (`SAU_PRO`). The
  transaction runs cursor `T001E11` and, if any row is found, raises `GXM_del("Profissionais")` /
  `CannotDeleteReferencedRecord`. `kind: referential` · `confidence: high` · src
  `onDeleteControls1E57 :1389-1404` · `target: #rejectsDeleteWhenReferencedByProfissional`.
  **Implementation note:** the referencing column is `SAU_PRO.ConClaCod` (Conversion.xml AttriId 238,
  TakesValueFrom SAU_PRO). SAU_PRO is **Wave 4 / un-migrated** → implement the guard as a native
  `COUNT(*) FROM SAU_PRO WHERE ConClaCod = ?` until SAU_PRO has an entity, then swap to a derived count.
- **R4** — Optimistic-concurrency check on UPDATE/DELETE: the row is re-read and `sigla`+`nome` are
  compared to the values held since load; a mismatch → `RecordWasChanged`, a DB lock → `RecordIsLocked`.
  `kind: concurrency` · `confidence: high` (rule) / `medium` (how to port — there is **no version
  column**, so this is a re-read-compare, not JPA `@Version`) · src `checkOptimisticConcurrency1E57
  :1185-1216` · `target: #detectsConcurrentModification` (optional; GeneXus framework behavior).
- **R5** — `sigla` and `nome` are **optional** (`AllowNulls=Yes`; `checkExtendedTable1E57` is empty —
  no required-field validation at the transaction level). `kind: validation (negative)` ·
  `confidence: high` that legacy allows null · src `Conversion.xml` (179/861 AllowNulls=Yes),
  `:1503-1511` · **but see open question 2** — a council without a sigla is questionable data quality.

## Open questions (resolve via KB/IDE / DB / product before `verified`)
1. **Auth:** legacy disables integrated security here. Confirm the role that should gate CRUD in the
   new app (proposing `SAUDE_CADASTRO` to match the SAU_ESP reference) and the exact permission name.
2. **Required `sigla`?** Legacy allows null sigla/nome. Decide whether the new app should require
   `sigla` (and/or `nome`) for data quality, or preserve the lax legacy behavior for parity.
3. **Live-DB confirmation:** verify actual PG column types/nullability match the reorg DDL
   (`smallint` / `VARCHAR(10)` / `VARCHAR(100)`) — introspection was not run (no DB reachable here).
4. **Delete-guard column:** confirm `SAU_PRO.ConClaCod` is the (only) referencing column; check
   whether other tables also reference `SAU_CONCLA`. Until SAU_PRO migrates, the guard is a native query.
