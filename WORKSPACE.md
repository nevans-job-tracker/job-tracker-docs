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

## Deploying

Deployment is deliberately manual — one deploy every few weeks does not earn a
pipeline. The ordering below matters, so follow it rather than improvising.

**Backend first**, because restarting it runs the migrations:

```bash
cd /opt/job-tracker-backend
git pull --ff-only origin main
git submodule update --init
.venv/bin/pip install -r requirements.txt    # only if requirements changed
sudo systemctl restart job-tracker-backend
```

The unit's `ExecStartPre` runs `alembic upgrade head`, so the schema is brought
forward as part of the restart. If a migration fails the service does not
start, which is the intended behaviour — a service running against a schema
behind its code is worse than one that is down.

**Then the frontend:**

```bash
cd /opt/job-tracker-frontend
git pull --ff-only origin main
git submodule update --init
npm ci
npm run build
```

nginx serves `dist/` straight off disk, so there is nothing to restart. Only a
change to `deploy/nginx.conf` needs `sudo nginx -t && sudo systemctl reload
nginx`.

**Then verify:**

```bash
/opt/job-tracker-backend/deploy/run-tests.sh
curl -s -o /dev/null -w '%{http_code}\n' http://localhost/api/health
```

The same script the nightly timer runs. It is safe against the live database —
see `REQUIREMENTS.md` §5 for why that is an assertion rather than a convention.

**Cold-load a deep URL** afterwards, e.g. `http://<server>/applications/1`
pasted into a fresh tab. Clicking around inside the app cannot detect a broken
SPA fallback, because the router handles in-app navigation without ever
reaching nginx.

## Restoring

Verified end to end on 2026-08-21 (KAN-19): **42 seconds**, from a backup the
timer produced, using only what would survive the loss of the machine.

```bash
/opt/job-tracker-backend/deploy/restore.sh
```

It takes the newest artifact from B2 unless given a filename, loads it into a
scratch database, and compares against the live one — row counts, the Alembic
revision, and spot-checks that read back the archived record and the contact
join rather than only counting rows.

**It asks you to type the passphrase, and will not read
`~/.config/job-tracker/backup.pass`.** That is deliberate and is the point of
the exercise. The scenario is that this machine is gone, so every off-site
artifact is unreadable unless the passphrase exists somewhere else. If the
decrypt fails using the copy from the password manager, *that failure is the
finding* — not a broken script.

This nearly went wrong for real: during KAN-18 the password-manager entry did
not save and the passphrase existed only on the machine being backed up. A
restore reading the server's copy would have passed happily in that state,
which is worse than not testing at all.

`sudo` is needed to create the scratch database, because the application user
is scoped to its own schema and cannot create another. That matches a real
recovery, where you would have root on a fresh machine.

**Repeat it after any schema change ships through Alembic.** A restore path is
only as good as the schema it restores into, and an old dump meeting a newer
schema is exactly the case that fails quietly.

> **A repeat is currently due.** The 42-second rehearsal above ran against the
> baseline schema. KAN-31 shipped `4500fe76cbd9` on 2026-08-21 — `date_applied`
> nullable and `interested` appended to the status enum — so the newest
> artifacts were dumped from a schema the rehearsal never restored into. This is
> exactly the trigger the paragraph above describes.

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
- **The backend test suite empties every table.** `tests/conftest.py` refuses
  to start unless the engine it received is SQLite, so a misconfigured run
  aborts rather than truncating live data. Do not remove that check — see
  `REQUIREMENTS.md` §5.
- **The frontend tests need a newer Node than the build does.** `jsdom@30`
  requires Node 22.22.2+ or 24.15+. Debian 12 ships 18, on which `npm ci` and
  `npm run build` both succeed and give no warning — then every test fails with
  `ERR_REQUIRE_ESM`. The server runs Node 24 from NodeSource.
- **The backend needs Python 3.10–3.12.** `pydantic==2.9.2` has no wheel for
  3.14 and fails to build from source, deep inside `pip install`, in a way that
  does not look like a version problem.
- **`/health` returns 200 on an un-migrated database.** It touches nothing, so
  a passing health check is not evidence the schema is present.
- **The app creates no schema.** `alembic upgrade head` is the only mechanism;
  starting against an un-migrated database fails on the first query that
  touches a table, by design.

## Current state

**The app is deployed and running** at `http://192.168.0.151/` — nginx serving
the built frontend, uvicorn on loopback behind it, MariaDB holding the schema.
Both suites run nightly on the server.

Backups run nightly to off-site object storage, encrypted before upload, and
**a restore has been verified end to end** — see Restoring above.

See [`ARCHITECTURE.md`](ARCHITECTURE.md) for the story-by-story breakdown.
Everything originally planned is done. Outstanding work is KAN-30, a set of
usability changes wanted after real use.
