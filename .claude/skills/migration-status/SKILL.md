---
name: migration-status
description: Show a dashboard of the GeneXus→Spring Boot migration — each transaction's wave, complexity, and current slice status (pending/specced/backend/tested/verified/cutover), with totals and the recommended next slice.
---

# migration-status

A read-only progress dashboard. No side effects.

## Steps
1. Read `BACKLOG.md` (waves + complexity) and every `slices/*.slice.md` (`status` + open_questions).
2. Build a table: `Wave | Transaction | Complexity | Status | Open questions | Blockers`.
   Status legend: `pending → specced → backend → tested → verified → cutover`.
3. Totals: count by status; % of transactions verified/cut over; count of unresolved open_questions
   (highlight regulatory/security ones).
4. **Recommend the next slice**: the lowest-wave `pending`/`specced` transaction whose `depends_on`
   are all `verified`+ — i.e. the next safe thing to work on. Note any slice blocked on an
   un-migrated dependency.
5. Print the dashboard. Suggest the command to run next (`/gx-extract` or `/migrate-slice`).
