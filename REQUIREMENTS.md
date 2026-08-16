# Job Tracker — Requirements

Functional requirements for the Job Tracker application. Architecture decisions
live in `CLAUDE.md`; this file covers what the app must *do*.

Status of each requirement is marked:

- **[built]** — implemented and working today
- **[gap]** — not implemented, and we want it
- **[open]** — needs a decision before it can be specified
- **[decided]** — decision made and recorded here

---

## 1. Purpose and users

Track personal job applications through their lifecycle, replacing an ad-hoc
spreadsheet. Single user (the owner), single machine. Not a multi-tenant
product, not shared with anyone else.

**[decided] Access scope.** LAN access is in scope now and does *not* require
authentication — development and daily use happen on the owner's own network.
Remote access (static IP + port forwarding) is a later intention, and is the
point at which authentication becomes mandatory. See §6.

**[decided] Mobile.** The app is used from a phone on the LAN, so responsive
layout is a real requirement rather than a nicety. This drives the screen design
in §4.4 and the column choices in §4.2.

---

## 2. Data model

An **application** represents one job applied to at one company.

| Field | Type | Required | Status |
|---|---|---|---|
| `id` | int | auto | [built] |
| `company` | string(255) | yes | [built] |
| `role_title` | string(255) | yes | [built] |
| `job_link` | string(1024) | no | [built] |
| `source` | string(255) | no | [built] — free text (LinkedIn, referral, …) |
| `location` | string(255) | no | [built] |
| `status` | enum | yes, defaults `applied` | [built] |
| `salary_min` | decimal(10,2) | no | [built] |
| `salary_max` | decimal(10,2) | no | [built] |
| `salary_currency` | string(10) | no, defaults `USD` | [built] |
| `date_applied` | date | yes | [built] |
| `notes` | text | no | [built] |
| `created_at` / `updated_at` | datetime | auto | [built] |

### Planned fields

Not yet built.

| Field | Type | Required | Status |
|---|---|---|---|
| `next_action` | string(255) | no | [decided] — what is owed next ("follow up", "take-home due") |
| `next_action_date` | date | no | [decided] — when it is owed; gives the list an actionable sort |
| `job_description` | text | no | [decided] — snapshot of the posting, which outlives the link |
| `archived_at` | datetime, nullable | no | [decided] — archive marker; `NULL` means active (§4.1) |

**[decided] No migration concern for any of this.** The server environment and
database do not exist yet, so the schema below is simply the schema we build.
Alembic (§5) remains worth having before the *first* change after real data
exists — it is not a prerequisite for the initial build.

### 2.1 Contacts

**[decided] Contacts live in their own table**, related many-to-one to
applications, so a single application can carry several people (e.g. recruiter,
hiring manager, referrer) each with their own details.

Each of the following is a separate column on the `contacts` table.

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | int | auto | |
| `application_id` | int, FK → `applications.id` | yes | |
| `name` | string(255) | yes | |
| `title` | string(255) | no | The contact's position — Manager, HR, Sr. Quality Engineer, … |
| `phone` | string(50) | no | Free text; formats vary (extensions, international) |
| `email` | string(255) | no | |
| `notes` | text | no | Free text for anything specific the contact mentioned |
| `created_at` / `updated_at` | datetime | auto | |

- **[decided]** `title` covers what was previously an open question about a
  `role` / `relationship` field. It is free text rather than an enum — job
  titles are too varied to constrain, and this is a personal reference field.
- **[decided]** Contacts follow their application. Archiving an application
  keeps its contacts intact, so unarchiving restores the full record. Since
  records are never purged (§4.1), no cascade-delete behaviour is needed.
- **[decided] Contacts are hard-deleted, and this is an expected use case.**
  Removing a contact from an application's detail screen is normal usage, not an
  edge case. The no-delete rule in §4.1 governs *applications*, which are
  history worth keeping; a contact is a detail *of* an application, and the
  application itself is preserved regardless. No archive mechanism for contacts.
- **[built]** Contacts are managed through endpoints nested under their
  application (`/applications/{id}/contacts/...`), and every lookup is scoped by
  `application_id` so a contact cannot be read or modified through another
  application's URL.
- **[built]** The list endpoint returns applications *without* contacts; only
  the detail endpoint embeds them. Loading contacts per row would mean one query
  per application on every list request.

### Field-level rules

- **[built]** `salary_min` must not exceed `salary_max`. Enforced in the route
  rather than the schema, because a PATCH may supply only one of the two and the
  rule has to hold against the **merged** result — lowering only `salary_max`
  can still invert the pair against the stored `salary_min`.
- **[built]** `job_link` is validated as an http(s) URL server-side, not just by
  the browser form's `type="url"`. The value is stored exactly as entered rather
  than pydantic's normalised form, so a pasted link comes back unchanged.
- **[built]** A future `date_applied` is **warned on, not rejected**. A future
  date is usually a typo (`2027` for `2026`, which sorts to the top and stays
  there), but logging an application about to be submitted is legitimate, so the
  input is still accepted. The warning appears beside the field in the form and
  is tied to it with `aria-describedby`; the API imposes no rule.
- **[built] Validation errors are rendered readably.** FastAPI reports schema
  failures as a *list* of objects, which renders as `[object Object]` if passed
  straight to `Error()`. The API client now formats those into
  `field: message`, and rules the API raises itself use a plain sentence.
- **[decided]** `source` stays **free text for now**. A controlled list is
  premature before the real values are known. Accepted consequence: values will
  fragment ("LinkedIn" vs "linkedin"), degrading filtering. Revisit once enough
  entries exist to see which sources are actually used — at which point the
  cleanup is a one-off normalisation.

---

## 3. Status lifecycle

Seven statuses: `applied`, `phone_screen`, `interview`, `offer`, `rejected`,
`ghosted`, `withdrawn`.

**[decided] Free assignment.** Any status may be set to any other status at any
time. There is no transition validation, no terminal states, and no required
ordering — `phone_screen` can be skipped, `rejected` can be reopened to
`interview`, and `ghosted` is set manually by the user.

This matches the current implementation, so no code change is needed. It is an
explicit decision rather than an oversight: for a single-user tool the owner is
the only actor, and enforcing a workflow on yourself adds friction without
preventing any real error.

Consequences:

- No backend transition validation will be added.
- No time-based automation sets `ghosted`; it is a manual judgement call.
- Any future reporting on the lifecycle (e.g. time-in-stage) cannot assume a
  monotonic progression through statuses.

---

## 4. Functional requirements

### 4.1 Managing applications

- **[built]** Create an application via a form; company, role title, and date
  applied are required, everything else optional.
- **[built]** Edit any field of an existing application.
- **Superseded** — the former delete action is gone, replaced by Archive below.
  There is no DELETE route for applications and no delete call in the API
  client; the route responds 405.
- **[built]** New applications default to status `applied` and today's date.
- **[built] Archive replaces Delete entirely.** There is no delete action in
  the UI. Archiving sets `archived_at`; the record is retained in full and can
  be unarchived at any time. Exposed as `POST /applications/{id}/archive` and
  `.../unarchive`; `archived_at` is read-only and cannot be set through PATCH.
  The list takes a `show` parameter of `active` (default), `archived`, or `all`.

  **Why archive rather than delete:** status records *what happened* to an
  application; archive records *whether it should still be in view*. These are
  independent — a recent rejection you are still following up on stays active,
  while an older one is archived. A status filter cannot express that intent,
  so archive is not a duplicate of `rejected` / `withdrawn`.

  - The list defaults to **Active** (`archived_at IS NULL`), with a control for
    **Active / Archived / All**. Three states rather than a show/hide toggle, so
    the archive can be reviewed on its own.
  - This control is independent of the status filter; both apply at once.
  - Archived applications keep their contacts, so unarchiving restores the whole
    record intact.
  - **[decided]** Records are never purged. Archived applications are retained
    indefinitely — acceptable at personal-tracker scale.
  - **[decided] No hard delete anywhere in the application.** A mistaken record
    is an edge case, and the detail screen (§4.4) allows editing essentially
    every field — so a wrong entry is corrected in place rather than removed.
    Duplicates or unwanted records are archived. This keeps one concept instead
    of two; hard delete can be added later if it ever proves necessary.
  - **[decided] No confirmation prompt on archive.** Confirmations exist to
    guard irreversible actions, and archiving is reversible in one click.
    Prompting would add friction to a frequent, cheap action. (A prompt *was*
    specified while this was a permanent delete — that rationale disappeared
    with the delete.)

### 4.2 Viewing and finding

- **[built]** Table view: company, role, location, status, salary, date applied,
  link to posting, row actions.
- **[decided]** The table is reduced to a mobile-viable column set. Eight columns
  do not fit a phone.

  | Column | Visibility |
  |---|---|
  | Company | Always |
  | Status | Always |
  | Next action | Always |
  | Date applied | Always; default sort |
  | Role title | Wider screens only |
  | Location | Wider screens only |
  | Salary | Wider screens only |
  | Source | Wider screens only — **[built]**, the first responsive column |

  **[decided] The narrow-screen breakpoint is 900px.** The target mobile device
  is an **iPhone 17 Pro**: 402 × 874 CSS pixels, device pixel ratio 3. That means
  402px wide in portrait and **874px in landscape** — so a conventional 768px
  breakpoint would correctly hide wide columns in portrait but expose them again
  the moment the phone is rotated. 900px clears the landscape width with margin.

  Trade-off: a desktop browser window narrower than 900px is also treated as
  narrow. That is the intended behaviour — the constraint is available width,
  not device class.

  Columns hidden below the breakpoint carry a `col-wide` class; the rest of the
  responsive set is applied under KAN-12 using the same mechanism.

  **[decided] Role title is kept but shown only on wider screens.** The exact
  advertised title is wanted for quick reference, but applications are
  effectively one role per company, so it is the column that gives up its place
  on a phone. This keeps `next_action` — the most actionable field — visible on
  mobile within the four-column budget.

  **Link and row actions are dropped** — both move to the detail screen. Rows
  become clickable (§4.4), so a per-row Delete button is a mis-tap hazard on
  touch.

- **[decided]** The mobile column budget is resolved by demoting Role rather
  than abandoning the table. Narrow screens show Company, Status, Next action,
  and Date applied — four columns, within budget, with the actionable field
  present.

  A stacked-card layout for narrow screens remains a reasonable fallback if four
  columns still prove cramped in practice, but is not planned.

- **[decided]** The list gains an **Active / Archived / All** control alongside
  the existing status filter, defaulting to Active (§4.1).
- **[built]** Sort by company, role, status, or date applied — click a header to
  toggle ascending/descending. Default: date applied, descending.
- **[built]** Free-text search across company, role title, and location,
  debounced 250ms.
- **[built]** Filter to a single status, or all statuses.
- **[built]** Result count displayed; empty state when no applications exist.
- **[gap]** Search does not cover `notes` or `source`. Notes are where the
  useful detail lives (recruiter names, interview feedback) and are currently
  unsearchable.
- **[gap]** Location and source are not sortable, though location is displayed.
- **[built]** Search, filter, and sort state **persists in the URL**, not in
  local storage. It survives a reload, works with the browser back button, and
  makes a filtered view linkable. URL state is also inspectable in a way local
  storage is not.
  - Parameters at their default value are omitted, so an unfiltered list is a
    bare `/` rather than a URL full of defaults.
  - Typing in the search box **replaces** the history entry; changing the status
    filter or sort **pushes** one. Otherwise every keystroke would become a
    separate Back step.
  - The detail screen's back link returns through history when the user arrived
    from within the app, so the list's filters are preserved. On a cold load
    (bookmark or refresh) there is no history to return to and it falls through
    to a plain `/`.

### 4.3 Pagination

- **[built]** The API supports `skip` / `limit` and returns a `total` count.
- **[gap]** **The frontend never sends them.** It relies on the API default of
  `limit=100` and renders whatever comes back. Past 100 applications, the table
  shows only the first 100 rows.

  The displayed count stays correct — `total` is computed before `offset`/`limit`
  is applied — so the symptom is a visible mismatch: the header reads "342
  applications" while 100 rows are listed. Wrong, but self-announcing rather
  than silent.

  This is the one gap with a real failure mode rather than a rough edge, and it
  gets worse the longer the tool is used successfully.

- **[decided]** Fix with a **"Load more" control**. The frontend tracks `skip`
  and appends results rather than replacing them, using the pagination the API
  already provides. Chosen over a paged table (more UI than a personal tracker
  needs) and over simply raising the limit (moves the ceiling instead of
  removing it).

### 4.4 Screens and navigation

**[decided] The app moves from one page to three routed screens.** This reverses
the "no routing library, everything is one page" decision in `CLAUDE.md`, which
must be updated to match.

| Screen | Route | Purpose |
|---|---|---|
| List | `/` | Sortable, searchable, filterable table (§4.2) |
| Detail | `/applications/:id` | Full details for one application, editable in place |
| New | `/applications/new` | Same layout as Detail, empty, for creating an entry |

- **[decided]** Rows in the list are clickable and navigate to the detail screen.
  This replaces the inline edit form and the per-row Edit button.
- **[decided]** The detail screen *is* the edit form — populated and saveable,
  rather than a read view with a separate edit mode. Fewer states, one component.
- **[decided]** The new-entry screen is the same component with no initial
  values, so both screens share look and behaviour by construction.
- **[decided]** Archive lives on the detail screen, keeping it away from touch
  targets in the list. Unarchive appears there too when viewing an archived
  application.
- **[decided]** A router (`react-router-dom`) is added rather than
  state-switched views, so each application has its own URL. Real URLs make the
  browser back button behave correctly, which matters most on mobile, and make
  a specific application bookmarkable.
- **[built]** No backend work is needed for the detail screen — the API already
  serves `GET /applications/{id}`.

---

## 5. Non-functional requirements

- **[built]** REST/JSON API with auto-generated interactive docs at `/docs`.
- **[built]** `/health` endpoint for service monitoring.
- **[built]** CORS restricted to origins configured via `CORS_ORIGINS`.
- **[built]** Frontend targets the API via `VITE_API_URL`.
- **[built]** Tables auto-created on startup via `Base.metadata.create_all`.
- **[decided] The frontend requires SPA fallback routing when served statically.**
  With client-side routes, a direct request for `/applications/10` — a bookmark,
  a refresh, or a shared link — hits the static server for a path that has no
  file. Whatever serves the build must rewrite unknown paths to `index.html`, or
  deep links 404 in production while working perfectly in the Vite dev server.
  Both candidates already documented in the frontend README satisfy this
  (nginx `try_files`, `serve -s`); the constraint is not to lose it when the
  serving stack is chosen under KAN-14.
- **[built] Automated tests.** Backend: pytest against throwaway SQLite, no
  MySQL required — `DATABASE_URL` is overridden before the app is imported, and
  each test starts from empty tables. Frontend: Vitest + Testing Library in
  jsdom. Run with `pytest` and `npm test` respectively.
  - Both suites write HTML coverage and result reports on every run
    (`htmlcov/`, `report.html`, `coverage/`, `test-results/`). All four are
    generated output and belong in `.gitignore` once the repos are under git.
  - **Coverage as measured:** backend 99% of statements — the only uncovered
    line is the MySQL URL branch, which tests never take by design. Frontend
    99%, covering routing, the API client, both page components, and all three
    UI components.
  - **[decided] Automating the test runs is deferred until after deployment**,
    at which point they run on deploy and nightly (cron or equivalent) on the
    server. Until then the suites are run by hand. Tracked with KAN-14 rather
    than KAN-11.
  - **The test suite destroys data, so the scheduled run must never touch the
    live database.** `tests/conftest.py` empties every table after each test.
    That is safe only because `DATABASE_URL` points at a throwaway SQLite file.
    A cron job or deploy hook that inherits the service's own environment —
    where `DATABASE_URL` or the `DB_*` settings point at production MySQL —
    would wipe the job search history. The scheduled run must set its own
    `DATABASE_URL` explicitly, and never reuse the service's `.env`.
- **[gap]** No schema migrations. Any column change requires manual SQL or
  dropping tables — destructive once real application data exists.
- **[decided] Backup is required.** The stated concern is a hard drive crash, so
  the backup must survive the loss of the machine — a local dump on the same
  disk does not satisfy this. Losing the data means losing the job search
  history, which is not reconstructable from any other source.
  - **[open — deliberately deferred]** Where backups go (external drive, another
    machine on the LAN, off-site/cloud) and on what schedule. Decision postponed
    until the server environment exists, since the options depend on it.
  - A restore must be tested, not assumed. An unverified backup is not a backup.
  - Tracked in KAN-10.

---

## 6. Deferred, with triggers

Not in scope now. Each has the condition that would pull it in:

### 6.1 Remote access — a blocking prerequisite, not a deferred item

The intended end state is remote access via static IP and port forwarding. That
step has a hard prerequisite worth stating plainly rather than filing as a
"nice to have":

**The API has no authentication of any kind today.** Port-forwarding it as-is
publishes full read, write, and delete access to every application record — and
the whole `/docs` interface — to anyone who finds the address. Authentication
and HTTPS are blocking conditions for remote exposure, not enhancements to
schedule afterwards.

**Authentication must be enforced at the API layer, not the frontend.** A login
screen in the React app is decorative: the FastAPI service is reachable
directly, so anything that only gates the UI leaves every endpoint open. Any
auth work must protect the API itself, with the frontend merely obtaining and
presenting a credential.

This does not affect LAN use, which is explicitly in scope without auth (§1).
The decision to defer auth is deliberate and revisitable — but it is tied to
access scope, so it must be revisited *before* the app is exposed remotely
rather than after.

### 6.2 Deferred items

| Item | Trigger |
|---|---|
| Authentication + HTTPS | Exposing the app beyond the LAN — static IP / port forwarding. Blocking, see §6.1 |
| File attachments (resume/cover letter per application) | Wanting per-application document history |
| Alembic migrations | First schema change after real data exists |
| Automated tests | Whenever change becomes risky enough to want a safety net |
| Controlled `source` list | Once source values fragment enough to hurt filtering |

---

## 7. Non-goals

Stated so they stop resurfacing:

- Multi-user support, sharing, or collaboration.
- Job board scraping or automatic import of postings.
- Email or calendar integration.
- Mobile app (responsive web is sufficient).
- Public hosting or deployment beyond the owner's own machine.
- Analytics/reporting dashboards beyond the basic result count.
