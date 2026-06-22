---
name: verify-parity
description: Run golden-master parity verification for a migrated slice against the running GeneXus app — capture legacy behavior, run the new endpoints, diff for business equivalence, and report divergences. The acceptance gate before cutting a route over to the new app.
---

# verify-parity <TRANSACTION>

Confirms the migrated slice behaves like legacy before cutover. Requires the slice's backend
generated and a reachable running GeneXus app + a non-production DB copy/snapshot.

## Usage
`/verify-parity SAU_ESP`

## Steps
1. Read the SLICE-SPEC `parity` block (legacy base URL, scenarios). Confirm with the user the
   legacy app URL and that a **non-production DB copy/snapshot** is available (refuse write-scenarios
   against production).
2. Start the new app (or point at a running instance) and obtain a JWT.
3. Spawn `parity-verifier` with the scenarios. It captures legacy results, runs the new endpoints,
   diffs for business equivalence, and saves fixtures under
   `backend/src/test/resources/parity/<slice>/`.
4. Persist the fixtures and the parity report (main thread). Update the SLICE-SPEC: list each
   scenario's verdict; set `status: verified` only if all scenarios are PARITY (or divergences are
   explicitly accepted as intended improvements, with justification recorded).
5. Report the verdict and any divergences as cutover blockers.

## Notes
- Reads/lists: compare result sets/values, ignore HTML chrome + locale formatting.
- Writes: compare resulting DB rows on the shared schema (most reliable equivalence check).
- Never commit real PHI into fixtures — synthesize/redact identifying fields.
