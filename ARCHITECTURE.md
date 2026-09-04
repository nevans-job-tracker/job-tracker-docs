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
- **The server exists** (KAN-21): an eMachines ET1810 from 2009 — single-core
  1.6 GHz Celeron 420 — running Debian 12 headless at a reserved
  `192.168.0.151`.

  Two upgrades before the install make it viable rather than merely
  sentimental: **RAM raised to 4 GB** (3.6 GiB usable) and the original
  spinning disk **replaced with an SSD**. Do not read "2009 desktop" as
  original spec — the performance is unremarkable in a good way because of
  those two changes, not despite the age.

  Both services plus MariaDB run comfortably. The single core is the only real
  constraint and is not a binding one at this scale: 43s to build the Python
  venv from scratch, 18.8s for a production Vite build.

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

**The app is deployed and in daily use** at `http://192.168.0.151/`, entered
from a phone on the LAN. Detail lives in Jira; this is the shape of it.

**Done epics:** KAN-6 Planning, KAN-8 Search & Navigation, KAN-9 Data Integrity
& Validation, KAN-11 Test Coverage, KAN-12 Detail & Entry Screens, KAN-13
Archive & Restore, **KAN-14 Server Environment & Deployment**.

**KAN-14 completed** across seven stories: nginx on a single origin (KAN-20),
Debian 12 on the eMachines at a reserved address (KAN-21), MariaDB with the
schema migrated (KAN-22), uvicorn under systemd on loopback (KAN-23), the built
frontend served with SPA fallback verified (KAN-24), a real session from an
iPhone (KAN-25), and both suites running nightly (KAN-26).

Three bugs came out of KAN-25 and were fixed in the same pass — the form and
filter row did not wrap on a phone, and the Applied date broke mid-value.
All three had one cause: `index.css` contained exactly one media query. The
table had been made responsive under KAN-12 and nothing else had.

**KAN-10 — Data Durability. Done.** Migrations own the schema, backups run
nightly off-site, and a restore has been verified end to end.

| Story | State |
|---|---|
| KAN-15 Alembic baseline | **Done** |
| KAN-16 Migrations own the schema | **Done** |
| KAN-17 Backup destination and schedule | **Done** — decided, see §5 |
| KAN-18 Automate the backups | **Done** — Backblaze B2, nightly, encrypted |
| KAN-19 Verify a restore | **Done** — 42s, from a timer-produced artifact |

**Everything originally planned is now done.** The remaining work is KAN-30, a
set of usability changes wanted after actually using the tracker.

**KAN-34** replaced three status formatters with the one label map the badge
already used. **KAN-31** made `date_applied` optional and added an
`interested` status, so a job can be tracked before it is applied for — and
carried the first Alembic revision that *alters* an existing table rather
than creating one. **KAN-36** shows salaries in thousands and drops the USD
suffix, and **KAN-33** put an Add control on the detail screen so entries can
be made in succession. **KAN-32** and **KAN-35** added a years-of-experience
minimum and a company size on Wellfound's bands, in one revision.

**KAN-30 is complete.** Since then: **KAN-37** made the restore rehearsal
record itself, **KAN-38** removed the free-text currency input, and
**KAN-39** added a CSV export of the filtered list. **KAN-40** stores the
cover letter as text, downloadable as HTML — the cheap half of the file
attachments §6.2 defers — and **KAN-41** converts an uploaded `.docx` into it
in the browser, which is the first runtime dependency added since the router.

**KAN-42** started recording every status change. Nothing reads it yet — it
shipped alone because history cannot be reconstructed afterwards, so the
recording is the only part with a deadline. A timeline and a status-over-time
graph are the stories it unblocks. **KAN-43** built the first of those: a
timeline on the detail screen, and the first thing that reads the history.

**The graph is built** (KAN-70) — the trigger below was met, at 44
applications outside `interested` against a threshold of 20. It is a stacked
area of applications per status per day on its own `/insights` route, hand-
rolled SVG reading the same `--badge-*` tokens the list does, and it adds
~13 KB rather than a charting library's share of the bundle.

**Two designs were rejected before that one**, and for the same reason. A
funnel and a time-in-stage bar both encode a claim about a *process* — "130
saved, 23 moved on, 1 offer" reads as a conversion rate — and while the tracker
is still mostly a shortlist that claim is false. A stacked area says only "this
is what was held on these days", which is true at any volume and improves on
its own as real transitions accumulate. §7 is amended rather than contradicted:
one reporting screen, not a dashboard.

**The left edge is a step, and the screen says so from a number rather than a
sentence** — the chart opens on 2026-08-23 holding 48 applications, the day
KAN-42's recording began, and rendering that as a day's activity would be the
same fiction the timeline's two notes exist to avoid. The note shrinks as
history accumulates because it is computed.

**Deploying it immediately found a defect the tests had not.** That opening
number counted history *rows* landing on the first day — 49 — where the chart
beside it drew 48 applications, because one of them moved twice that day. A
note contradicting the picture next to it is worse than no note, so it is read
off the first day's snapshot instead, where the two cannot disagree. Only real
data had that shape; every fixture written for it did not.

**Shipping it also caught a layout regression it caused** — a fourth control in
the list header, at 455px against the 402px an iPhone has in portrait, so
"+ Add application" left the screen entirely. Found by measuring the phone
breakpoint in a browser, which is the one thing jsdom can never check. The fix
is the shrink rules, and they are easy to get backwards: the container has to
be allowed to narrow so its children have a reason to wrap, while the controls
inside must not, or they squeeze into mis-tap-sized targets instead.

*The original deferral, kept because the trigger is the argument:*

**The graph was deliberately deferred, with a measurable trigger**
(KAN-61). Measured on 2026-08-30: 128 applications, 132 history rows, but
only **4 real transitions** — everything else is a creation stamp — and 122
of the 128 still sit at `interested`. A status-over-time chart is a
horizontal line with a few pixels of movement at the end. The tracker is
being used as a shortlist rather than a pipeline.

The trigger was one query rather than a judgement call: build it when
`SELECT COUNT(*) FROM applications WHERE status <> 'interested'` reaches 20.
Deliberately not time-based — weeks passing does not put data in the table,
applying for jobs does. It was met five days later.

Note the KAN-42 argument does *not* carry over. That recording shipped ahead
of anything reading it because history cannot be reconstructed afterwards,
which is a real deadline. A graph has none: built later it renders the same
rows plus everything since. KAN-61 also settles the design in advance — a
separate route, hand-rolled SVG rather than a charting library — and records
that §7 needs amending, since reporting is still listed there as a non-goal.

**KAN-44** added dark mode. The toggle was the small half; the work was turning
59 hardcoded colours into role-named tokens so the stylesheet could be themed
at all.

**KAN-45** made the job posting openable — an icon column in the list and a
control on the detail screen. The stored `job_link` had never been reachable
from anywhere in the UI, and §4.2 had recorded it moving to the detail screen
when the column was dropped, which never happened.

**KAN-46** stopped salary ranges wrapping at the en-dash on a wide desktop.
The auto table layout was sizing the column to the widest unbreakable run
rather than the whole string, so it wrapped with width to spare and widening
the window never helped. One `white-space: nowrap`, shared with the rule
`.col-date` already carried for the identical reason.

**KAN-49** consolidated every human-readable label map into `labels.js`, which
the frontend's own `CLAUDE.md` had said to do once a third one appeared.
**KAN-50** and **KAN-51** then added five columns in one revision: `pay_period`
so an hourly rate stops being told apart from a salary by magnitude alone,
`employment_type` and `contract_term_months`, and a `hours_per_week_min`/`_max`
pair. Employment type took Location's place in the list — the search is
effectively all-remote, so that column said "Remote" on nearly every row.

**KAN-52 is the interesting one, and it is not in these repos.** A fourth
repo — see `WORKSPACE.md` — scrapes postings into the API, and it turned out to
detect hourly pay already, then discard it with a warning, because the schema
had nowhere to put it. KAN-50 removes the reason that workaround exists.

**KAN-47** put required years of experience in the list as a sortable column.
No backend work — the route's `sort_by` whitelist already permitted it and the
NULL-sorts-greatest rule is generic. **KAN-48** added a clear button to the
search field, and corrected §4.2, which had been describing a three-field
search while the code had searched five for some time.

**KAN-56** added a source filter to the list. Its options are read from the
data rather than hard-coded, which turns §2's warning that free-text sources
"will fragment" into something the dropdown surfaces every time it is opened
rather than something that quietly degrades filtering.

**KAN-57** added a `posting_closed` status, for an ad that was pulled or
filled. `rejected` was the nearest and asserts a decision nobody made — often
the application was never sent. The label and badge were the whole frontend
change: the filter, timeline, export and form dropdown all read
`STATUS_LABELS`, so they picked it up for free, and the parametrised tests
over that map began covering it automatically.

**KAN-58** added Save and close, and turned the back link into a control
worth aiming at. The colour question it raised was answered with a hierarchy
rather than three colours — and the primary slot went to the *new* button,
because visual weight should follow how often something is used rather than
which control existed first.

**KAN-59** put a status dropdown in the list on wide screens. Performance was
the stated worry and was not the issue — 450 option elements against a table
already rendering ~500 cells. The real constraint was §4.2's mis-tap rule,
which KAN-45's link escaped and a data-changing control does not, so the
phone keeps the badge. Shipping it also introduced a 26px narrow-screen
regression from a `nowrap` that protected against something a `<select>`
cannot do; caught by measuring the phone breakpoint afterwards, which is the
one thing jsdom can never check.

**KAN-60** stopped the whole row opening the detail screen — Company and Role
are links now, and the row is inert. Asked for as "Role only", which would
have left a phone with no way in at all, Role being `col-wide`. The change
deleted three guards that existed solely because rows were clickable, and
replaced a `tabIndex` div with real anchors, so middle-click and the keyboard
work by construction rather than by handler.

**KAN-68** put "how long ago it was added" in the list, as a count of days
rather than a date — the question asked of `created_at` is age, and a date
makes the reader do the arithmetic. Frontend only: the column was already in
the list response and already a permitted sort key.

**KAN-69** fixed the extension reading pay wrongly, from two reported
postings that turned out to share only the symptom. Four defects: the pattern
required a literal `$` so `USD 99,000 - 128,500` was invisible; it allowed no
letters before the closing figure so `R$135,253.01` broke the range; the K/M
suffix ate the following word's first letter, turning "a $5,000 **m**ay
apply" into five billion; and the LinkedIn adapter preferred the top card,
which often carries LinkedIn's own estimate rather than the employer's
figure.

The third of those was found by a fixture written for something else, and is
the one that had never been reported — which is the argument for varying
test data beyond the examples that prompted the work.

**KAN-72** made Pay sortable by either end of the range, and it is the first
sort that does not order the stored column. Two pay periods share one pair of
columns, so sorting the number segregates the list rather than ordering it —
22 of 140 rows are hourly, so descending gave 118 salaries and then every
rate, with a $120/hr contract below a $60k job. The ORDER BY multiplies an
hourly rate by 2080.

**The KAN-50 precedent points the other way and does not apply**, which is
worth being explicit about. That story rejected a `salary_min < 1000 =>
hourly` backfill because it would have written a wrong fact into the
database permanently. This assumption reaches the ORDER BY and nothing else:
no stored value changes, the column still reads `86/hr`, and turning it off
is a one-line revert. Where an assumption *lands* is what decides whether it
is acceptable, not how plausible it is.

It is still an assumption — a contract is the case where 2080 is least
likely to hold — so the result count names the multiplier while a pay sort is
running and says nothing otherwise. **KAN-73** made the list's posting link
36px square instead of a ~20×16 glyph, matching the figure KAN-58 set for the
back link.

## Testing

Both suites run **nightly on the server** via a systemd timer (KAN-26), and by
hand during development:

```bash
cd job-tracker-backend && pytest        # 250 tests, 99% statements
cd job-tracker-frontend && npm test     # 550 tests, 99% statements, 100% functions
```

The backend suite runs against throwaway SQLite, so no database server is
needed. **It empties every table**, and `conftest.py` now refuses to start
unless the engine it received is actually SQLite — see `REQUIREMENTS.md` §5.

Two runtime constraints, both of which bite as confusing errors rather than
version complaints:

- **Python 3.10–3.12.** `pydantic==2.9.2` has no wheel for 3.14 and fails to
  build from source, deep inside `pip install`.
- **Node 22.22.2+ or 24.15+ for the *tests*.** `jsdom@30` declares that range
  and requires `require(esm)` support. The *build* is happy on Node 18, so
  Debian 12's packaged Node passes `npm ci && npm run build` and then fails
  every frontend test with `ERR_REQUIRE_ESM`. The server runs Node 24 from
  NodeSource — the one third-party apt repository in the deployment, accepted
  because Debian offers nothing newer and Node 18 is EOL.

It builds its schema by running the migrations, not `create_all`, so a green
suite also means the revisions apply and reverse cleanly. A revision that does
not apply fails at session setup rather than inside a test.
