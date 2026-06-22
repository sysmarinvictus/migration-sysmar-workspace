---
name: dev-stack
description: Start/stop the NEW app's local dev stack (PostgreSQL + Spring Boot backend on :8090 + React/Vite frontend on :5173) with one command. Wraps receituario-modern/dev-up.sh and dev-down.sh. The legacy GeneXus app stays on :8080 (managed separately). Use to run and interact with the migrated app in a browser.
---

# dev-stack [up|down|status] [snapshot|local] [--db]

Brings the migrated application up (or down) so it can be used in a browser. The backend runs on
**:8090**, the frontend (Vite) on **:5173**; the legacy GeneXus app keeps **:8080** and is never
touched. Login (dev stub, `local` profile): **`admin` / `admin123`**.

## Usage
```
/dev-stack up              # DB = live non-prod snapshot (host.docker.internal) — real data (default)
/dev-stack up local        # DB = local Docker postgres (fresh, Flyway-built, empty sandbox)
/dev-stack status          # report what's running (ports/PIDs/health)
/dev-stack down            # stop backend + frontend (leaves any local Docker DB up)
/dev-stack down --db       # also stop the local Docker postgres
```
Default when no arg: `up snapshot`.

## Steps
1. **Resolve the action** from the argument (`up`/`down`/`status`; DB mode `snapshot`|`local`;
   `--db`). Default to `up snapshot`.
2. **up:** run `./dev-up.sh <snapshot|local>` from `/mnt/s/projetos/receituario-modern/` **in the
   background** (it starts long-lived servers). The script is idempotent — it first stops any stack
   it previously started, then: checks/starts the DB, builds the jar if missing, installs frontend
   deps if missing, starts backend (:8090) + frontend (:5173), and health-checks both.
   - Because the servers are nohup-detached, the script returns once both pass health checks.
     Confirm readiness independently (don't trust buffered stdout): `curl` `localhost:5173/` (expect
     200) and `POST localhost:8090/auth/login {admin/admin123}` (expect 200).
3. **down:** run `./dev-down.sh [--db]`. It stops via PID files in `.devstack/`, with a safe
   `pkill -f 'receituari[o]-backend.jar'` fallback (the char-class prevents the command matching its
   own shell — a plain `receituario-backend.jar` pattern self-matches and kills the shell).
4. **status:** report `pgrep -fc 'receituari[o]-backend.jar'`, the vite process, and the two health
   checks; tail `.devstack/*.log` on failure.
5. **Report** the URLs (open **http://localhost:5173/**, or the WSL `Network:` URL if the Windows
   browser can't resolve `localhost`), the login, the DB mode, and where the logs live.

## Notes
- **snapshot mode** uses the live non-prod copy (real data; **writes persist to the shared
  snapshot** — the parity baseline). It reads `DB_PASSWORD` literally from `backend/.env` (the value
  starts with `$`, so it must NOT be shell-sourced), runs the backend Flyway-off + `ddl-auto=none`,
  and verifies the snapshot is reachable on `host.docker.internal:5432` first.
- **local mode** runs `docker compose up -d postgres` (creds `saude`/`change_me_local_only`, db
  `saude-mandaguari`) and lets Flyway build the schema — a safe, empty sandbox (no real PHI).
- Real snapshot data contains **PHI** and **CHAR columns render space-padded** (cosmetic).
- Login is the temporary dev stub (single admin) until the Wave-0 auth slice replaces it.
- Logs: `receituario-modern/.devstack/backend.log` and `frontend.log`. PID files live there too.
- Toolchain: JDK 21 + Maven + Docker + Node (repo targets Node 20+; runs on 18 with warnings).
