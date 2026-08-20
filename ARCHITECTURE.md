# Job Tracker — Architecture & Project Context

Summary of decisions made while scaffolding this project, for reference in
future sessions (e.g. Claude Code) that won't have the original chat history.

**This document is shared by both code repos** and lives in
[`job-tracker-docs`](https://github.com/nevans-job-tracker/job-tracker-docs),
included in each as a submodule at `docs/`. Edit it here — never in a
consuming repo's checkout.

**Functional requirements live in [`REQUIREMENTS.md`](REQUIREMENTS.md).** This
file covers architecture and how the project is put together; that file covers
what the app must do, and is the authority where the two overlap.

**Repo-specific design notes live in each repo's own root `CLAUDE.md`** —
backend ORM and endpoint choices in `job-tracker-backend/CLAUDE.md`, React and
routing choices in `job-tracker-frontend/CLAUDE.md`.

**Work is tracked in Jira**, project `KAN` on `job-tracker.atlassian.net`. The
Atlassian MCP server is configured in `.mcp.json` (Jira scopes only — no
Confluence or Bitbucket access).

## What this is

A personal web app to track job applications: company, role, job link, source,
location, status, salary range, date applied, notes, next action, and contacts.

## Architecture decisions

- **Two separate repos**, deployed independently:
  - `job-tracker-backend` — FastAPI REST API
  - `job-tracker-frontend` — React (Vite) single-page app
- **Shared documentation** lives in a third repo, `job-tracker-docs`, consumed
  by both as a git submodule. This keeps one source of truth for requirements
  and architecture without copying files between repos.
- **Database:** MariaDB (KAN-22), reached through the MySQL wire protocol.

  Originally specified as MySQL. Changed when the server was actually built:
  Debian ships MariaDB in its own archive, while Oracle MySQL needs a
  third-party APT repository and GPG key that then has to be maintained. On a
  machine whose whole point is running unattended, staying inside Debian's own
  security-update channel is worth more than matching the original name.

  The application is genuinely indifferent. `mysql+pymysql://` connects
  unchanged, nothing in the schema is MySQL-specific, and the Alembic baseline
  applied to MariaDB 10.11 with autogenerate then reporting **no drift** — the
  strongest available evidence that the models and the deployed schema agree.
- **Deployment target:** a single local Linux machine the user owns. Both
  frontend and backend run on that same machine, as separate services.
- **The server exists** (KAN-21): an eMachines ET1810 — single-core 1.6 GHz
  Celeron 420, 3.6 GiB RAM — running Debian 12 headless at a reserved
  `192.168.0.151`. Deliberately modest hardware; it comfortably runs both
  services plus MariaDB.

## Why FastAPI over Flask

Chosen because the frontend is a separate React app calling a JSON API.
FastAPI's built-in Pydantic validation and auto-generated interactive docs
(`/docs`) save time when building and testing that API surface, versus
Flask's more manual setup for validation/docs.

## Why two repos instead of one

Explicit user preference: independent update/deploy cycles for frontend and
backend, rather than a single app serving its own templates.

The docs repo is a third repo but not a third *deployment* — it is never
deployed, and its existence does not affect the independent release cycles
that motivated the split.

## Deployment notes

**Decided (KAN-20): nginx in front of both, on a single origin.**

```
phone / laptop ──▶ nginx :80 ──┬──▶ /        static files from dist/
                               │             (try_files → index.html)
                               └──▶ /api/    proxy to 127.0.0.1:8000
                                                  │
                                             uvicorn under systemd
```

- **Backend:** `uvicorn` under systemd, bound to `127.0.0.1:8000` — not
  reachable from the LAN except through nginx.
- **Frontend:** `npm run build`, and nginx serves `dist/` directly. The `serve`
  package is not used.
- **`VITE_API_URL=/api`** — a relative path, not an absolute origin.
- **`CORS_ORIGINS` is not exercised by this deployment**, because everything is
  same-origin. It still matters in development, where Vite serves on `:5173`
  and calls the API on `:8000`; those are different origins and CORS applies as
  before.

### Why a reverse proxy earns its place here

The question was whether a proxy is overkill for a single-user LAN app. Four
things decided it:

1. **It replaces a component rather than adding one.** Something has to serve
   `dist/` either way. nginx does that *and* proxies; `serve` only does the
   first. Two supervised services in both designs — but with nginx, Node is a
   build-time dependency only, not a runtime one.

2. **Same-origin removes CORS as a failure mode.** A `CORS_ORIGINS` mismatch
   fails *only in the browser* while `curl` against the API works perfectly,
   which is a genuinely confusing thing to debug. Not having the mechanism in
   the path removes the whole class.

3. **A relative `VITE_API_URL` decouples the build from the address.** Vite
   inlines env vars at build time, so with a two-port design the server's IP is
   baked into the bundle and changing the address means *rebuilding the
   frontend*. With `/api` it does not.

4. **It is where TLS and auth go later.** §6.1 makes remote access conditional
   on authentication at the API layer. A proxy is the natural place to
   terminate TLS and add a first gate, so choosing it now makes that work
   additive instead of a re-architecture.

The cost is one system package and one config file. The only thing the
two-port design wins is that a dead static server would leave the API up —
worth nothing here, since the app is unusable without its UI.

### Constraints this imposes

- **SPA fallback via `try_files $uri $uri/ /index.html`.** Client-side routing
  means a cold request for `/applications/10` has no file on disk. Without the
  rewrite, deep links 404 in production while working perfectly in the dev
  server. See `REQUIREMENTS.md` §5.
- **The `/api/` prefix is stripped at the proxy.** A trailing slash on
  `proxy_pass` (`proxy_pass http://127.0.0.1:8000/;`) maps `/api/applications`
  to `/applications`, so the FastAPI routes stay unprefixed and need no
  `root_path`.
- **`/docs` is reachable at `/api/docs` on the LAN.** Acceptable — LAN use
  without auth is explicitly in scope (§1) — but it is part of what must not be
  exposed when remote access is considered (§6.1).

## Security posture

- **No authentication of any kind.** This is fine for LAN use, which is
  explicitly in scope.
- **Remote access is blocked on adding auth.** The intended end state is remote
  access via static IP and port forwarding. Exposing the API as-is would publish
  full read/write access to every record, plus `/docs`, to anyone who finds the
  address. Auth must be enforced at the API layer — a login screen in React is
  decorative while FastAPI is directly reachable. See `REQUIREMENTS.md` §6.1.

## Current state and next steps

The app is feature-complete for its intended use and is **mid-deployment** —
the server is built and the database is live, but nothing is being served yet.
Detail lives in Jira; this is the shape of it.

**Every decision is made.** What remains is wiring the two services up on a
machine that now exists.

**Done epics:** KAN-6 Planning, KAN-8 Search & Navigation, KAN-9 Data Integrity
& Validation, KAN-11 Test Coverage, KAN-12 Detail & Entry Screens, KAN-13
Archive & Restore.

**KAN-10 — Data Durability.** Migrations half complete; backup half blocked.

| Story | State |
|---|---|
| KAN-15 Alembic baseline | **Done** |
| KAN-16 Migrations own the schema | **Done** |
| KAN-17 Backup destination and schedule | **Done** — decided, see §5 |
| KAN-18 Automate the backups | Blocked on the server |
| KAN-19 Verify a restore | Blocked on KAN-18 |

**KAN-14 — Server Environment & Deployment.** Under way.

| Story | State |
|---|---|
| KAN-20 Choose the serving stack | **Done** — nginx, single origin |
| KAN-21 Provision the Linux server | **Done** — Debian 12 at `192.168.0.151` |
| KAN-22 Database, user, schema | **In progress** — MariaDB live, schema migrated |
| KAN-23 Backend as a service | Next |
| KAN-24 Build and serve the frontend | After KAN-23 |
| KAN-25 Verify from a phone on the LAN | After KAN-24 |
| KAN-26 Automate the test runs | Unblocked — can run in parallel |

**The next step is KAN-23**: uvicorn under systemd, bound to `127.0.0.1:8000`,
with `alembic upgrade head` in the deploy path.

### Written ahead of the machine

Committed before the server existed, so the remaining stories are copy rather
than compose:

- `job-tracker-frontend/deploy/nginx.conf` — the site config, both load-bearing
  lines commented with why they matter.
- `job-tracker-frontend/.env.production` — `VITE_API_URL=/api`, loaded
  automatically by `vite build`.
- Backend README §4 — the systemd unit, binding `127.0.0.1`.
- Backend README §7 — Alembic commands and the autogenerate caveats.

## Testing

Both repos have suites, run by hand (automation is KAN-26, now unblocked):

```bash
cd job-tracker-backend && pytest        # 109 tests, 99% statements
cd job-tracker-frontend && npm test     # 137 tests, 99% statements, 100% functions
```

The backend suite runs against throwaway SQLite via a `DATABASE_URL` override,
so no MySQL is needed. **It empties every table** — never point it at real data.
`pydantic==2.9.2` has no wheel for Python 3.14; use 3.10–3.12.

It builds its schema by running the migrations, not `create_all`, so a green
suite also means the revisions apply and reverse cleanly. A revision that does
not apply fails at session setup rather than inside a test.
