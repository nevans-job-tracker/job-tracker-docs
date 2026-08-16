# Job Tracker — Workspace Layout

How the three repos sit together on a working machine, and the things that bite
before you have touched anything.

This file is the canonical copy. The workspace root holds a short `CLAUDE.md`
stub that imports it — the root is not a git repository, so the stub cannot be
versioned, but everything of substance lives here.

## The three repos

Cloned side by side, the working directory looks like this:

```
job-tracker/                 <- not a repo; local workspace only
├── CLAUDE.md                <- thin stub, imports this file
├── .mcp.json                <- Atlassian MCP config
├── job-tracker-backend/     <- FastAPI REST API
│   └── docs/                <- submodule → job-tracker-docs
├── job-tracker-frontend/    <- React (Vite) single-page app
│   └── docs/                <- submodule → job-tracker-docs
└── job-tracker-docs/        <- this repo: shared requirements and architecture
```

| Repo | What it is |
|---|---|
| [job-tracker-backend](https://github.com/nevans-job-tracker/job-tracker-backend) | FastAPI REST API |
| [job-tracker-frontend](https://github.com/nevans-job-tracker/job-tracker-frontend) | React (Vite) single-page app |
| [job-tracker-docs](https://github.com/nevans-job-tracker/job-tracker-docs) | Shared requirements and architecture |

Clone the code repos with `--recurse-submodules`, or `docs/` arrives empty. If
you already cloned without it: `git submodule update --init`.

## Where documentation lives

**This repo is the single source of truth** for anything spanning both code
repos — [`ARCHITECTURE.md`](ARCHITECTURE.md) and
[`REQUIREMENTS.md`](REQUIREMENTS.md). Both code repos mount it at `docs/`, so
the same files also appear at `job-tracker-backend/docs/` and
`job-tracker-frontend/docs/`.

Repo-specific design notes are deliberately *not* shared. They live in each
code repo's own root `CLAUDE.md`.

After changing a shared doc, publish it and then bump each consumer's pointer:

```bash
cd job-tracker-docs && git commit -am "Update requirements" && git push
cd ../job-tracker-backend && git submodule update --remote docs && git commit -am "Bump docs" && git push
cd ../job-tracker-frontend && git submodule update --remote docs && git commit -am "Bump docs" && git push
```

A stale pointer is harmless — it still resolves to a valid version. Bump when
the consuming repo needs the newer text.

## Branches

Both code repos use `main` and `develop`, with `develop` active. `main` is kept
level with `develop` by fast-forward — no merge commits, linear history:

```bash
git checkout main && git merge --ff-only develop && git push origin main && git checkout develop
```

This repo has only `main`. A docs repo does not need a staging branch, and the
submodules track `main`.

## Work tracking

Jira project `KAN` on `job-tracker.atlassian.net`. The Atlassian MCP server is
configured in the workspace's `.mcp.json` (Jira scopes only — no Confluence or
Bitbucket).

## Gotchas worth knowing before touching anything

None of these are discoverable from reading the code, and each has already cost
someone time.

- **Never edit files under a code repo's `docs/`.** It is a detached-HEAD
  submodule checkout; edits there are easy to make by accident and easy to
  lose. Edit this repo instead.
- **The backend test suite empties every table.** It is safe only because
  `tests/conftest.py` hardcodes a throwaway SQLite path. Never let it inherit
  an environment pointing at a real database — see `REQUIREMENTS.md` §5.
- **The backend needs Python 3.10–3.12.** `pydantic==2.9.2` has no wheel for
  3.14 and fails to build from source, deep inside `pip install`, in a way that
  does not look like a version problem.
- **`/health` returns 200 on an un-migrated database.** It touches nothing, so
  a passing health check is not evidence the schema is present.
- **The app creates no schema.** `alembic upgrade head` is the only mechanism;
  starting against an un-migrated database fails on the first query that
  touches a table, by design.

## Current state

See [`ARCHITECTURE.md`](ARCHITECTURE.md) for the story-by-story breakdown. In
short: **every design decision is made, and what remains needs hardware that
does not exist yet.** The next step is KAN-21, provisioning the Linux server.

The app has never been deployed and has only ever run against a local dev
database.
