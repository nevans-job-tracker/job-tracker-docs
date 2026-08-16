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
- **Database:** MySQL.
- **Deployment target:** a single local Linux machine the user owns. Both
  frontend and backend run on that same machine, as separate services.
- **This server does not exist yet.** Standing it up is tracked in KAN-14.

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

Nothing here is settled. The serving stack is deliberately an open choice,
to be decided in a story under KAN-14.

- Candidates for the backend: `uvicorn`, wrapped in a systemd service.
- Candidates for the frontend: build static files with `npm run build`, then
  serve via nginx or the `serve` package.
- Whatever origin the frontend ends up served from needs to be added to the
  backend's `CORS_ORIGINS`.

## Security posture

- **No authentication of any kind.** This is fine for LAN use, which is
  explicitly in scope.
- **Remote access is blocked on adding auth.** The intended end state is remote
  access via static IP and port forwarding. Exposing the API as-is would publish
  full read/write access to every record, plus `/docs`, to anyone who finds the
  address. Auth must be enforced at the API layer — a login screen in React is
  decorative while FastAPI is directly reachable. See `REQUIREMENTS.md` §6.1.

## Current state and next steps

The app is feature-complete for its intended use and has never been deployed —
it has only ever run against a local dev database. Work is tracked as epics in
Jira rather than listed here.

**Done:** KAN-6 Planning, KAN-8 Search & Navigation,
KAN-9 Data Integrity & Validation, KAN-11 Test Coverage,
KAN-12 Detail & Entry Screens, KAN-13 Archive & Restore.

**Outstanding:**

| Epic | Covers |
|---|---|
| KAN-10 | Data Durability — Alembic, backup and restore |
| KAN-14 | Server Environment & Deployment |

## Testing

Both repos have suites, run by hand (automation is deferred to KAN-14):

```bash
cd job-tracker-backend && pytest        # 109 tests, ~99% coverage
cd job-tracker-frontend && npm test     # 116 tests, ~99% coverage
```

The backend suite runs against throwaway SQLite via a `DATABASE_URL` override,
so no MySQL is needed. **It empties every table** — never point it at real data.
`pydantic==2.9.2` has no wheel for Python 3.14; use 3.10–3.12.
