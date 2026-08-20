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

**[built]** — every field below exists in `app/models.py`, listed in
declaration order.

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | int | auto | Primary key, indexed |
| `company` | string(255) | yes | Indexed — it is the default search and sort target |
| `role_title` | string(255) | yes | |
| `job_link` | string(1024) | no | Validated as an http(s) URL server-side |
| `source` | string(255) | no | Free text (LinkedIn, referral, …) |
| `location` | string(255) | no | |
| `status` | enum | yes, defaults `applied` | Seven values, freely assignable — §3 |
| `salary_min` | decimal(10,2) | no | Must not exceed `salary_max` |
| `salary_max` | decimal(10,2) | no | |
| `salary_currency` | string(10) | no, defaults `USD` | |
| `date_applied` | date | yes | A future date warns rather than rejects |
| `notes` | text | no | |
| `next_action` | string(255) | no | What is owed next ("follow up", "take-home due") |
| `next_action_date` | date | no | When it is owed; gives the list an actionable sort |
| `job_description` | text | no | Snapshot of the posting, which outlives the link |
| `archived_at` | datetime | no | Archive marker, indexed; `NULL` means active — §4.1 |
| `created_at` / `updated_at` | datetime | auto | Server-side defaults |

**This table is the content of the Alembic baseline revision** (§5), and it is
now live on the server as MariaDB 10.11 (KAN-22). Nothing was migrated *from* —
the baseline was the starting point, and every change after it ships as its own
revision. There is no `create_all` fallback: the schema arrives by
`alembic upgrade head` or not at all.

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
- **[built]** Sort by company, role, location, source, status, next action
  date, or date applied — click a header to toggle ascending/descending.
  Default: date applied, descending. Salary is the one displayed column that is
  not sortable.

  The API accepts a wider set than the table exposes — `salary_min`,
  `salary_max`, and `created_at` are permitted too — and rejects anything else
  with a 422. That whitelist is a security boundary, not a convenience:
  `crud.list_applications` resolves the column with `getattr`, so the pattern
  on the route is the only thing preventing an arbitrary attribute lookup.
- **[built]** Free-text search across company, role title, and location,
  debounced 250ms.
- **[built]** Filter to a single status, or all statuses.
- **[built]** Result count displayed; empty state when no applications exist.
- **[gap]** Search does not cover `notes` or `source`. Notes are where the
  useful detail lives (recruiter names, interview feedback) and are currently
  unsearchable.
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
- **[built]** **The frontend paginates with a "Load more" control.** It requests
  50 rows at a time, tracks `skip`, and *appends* each page rather than
  replacing the previous one.

  - The header reads `Showing 50 of 120 applications` while more remain, and
    falls back to a plain `120 applications` once everything is loaded.
  - The control is labelled with the remainder — `Load more (70 remaining)` —
    and disappears when nothing is left to fetch.
  - `total` is computed before `skip`/`limit` is applied, so the count is the
    true total rather than the number currently rendered.

  Chosen over a paged table (more UI than a personal tracker needs) and over
  simply raising the limit (moves the ceiling instead of removing it).

  This closes what was previously the one gap with a real failure mode: the
  frontend used to send neither parameter, relying on the API default of
  `limit=100`, so past 100 applications the table silently showed only the
  first 100 rows while the header reported the true count.

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
- **[built] Schema is owned by Alembic alone.** The app creates nothing at
  startup — the `Base.metadata.create_all` call is gone — so `alembic upgrade
  head` must run before the service starts, and on every deploy carrying a new
  revision. Starting against an un-migrated database fails on the first query
  that touches a table, which is deliberate: a service that silently creates
  what it is missing is a service whose schema nobody can reason about.
  - `/health` still answers `200` on an un-migrated database, since it touches
    nothing. A passing health check is not evidence the schema is present.
- **[decided] The frontend requires SPA fallback routing when served statically.**
  With client-side routes, a direct request for `/applications/10` — a bookmark,
  a refresh, or a shared link — hits the static server for a path that has no
  file. Whatever serves the build must rewrite unknown paths to `index.html`, or
  deep links 404 in production while working perfectly in the Vite dev server.
  The serving stack is now decided (KAN-20): nginx, providing the fallback with
  `try_files $uri $uri/ /index.html`. The constraint survives the decision — it
  is a property of the nginx config, so it can still be lost by editing that
  file carelessly, and it is not visible from clicking around the app. Only a
  *cold* load of a deep URL exercises it.
- **[built] Automated tests.** Backend: pytest against throwaway SQLite, no
  MySQL required — `DATABASE_URL` is overridden before the app is imported, and
  each test starts from empty tables. Frontend: Vitest + Testing Library in
  jsdom. Run with `pytest` and `npm test` respectively.
  - Both suites write HTML coverage and result reports on every run
    (`htmlcov/`, `report.html`, `coverage/`, `test-results/`). All four are
    generated output, and all four are gitignored.
  - **Coverage as measured:** backend 109 tests, 99% of statements — the only
    uncovered line is the MySQL URL branch, which tests never take by design.
    Frontend 137 tests, 99% of statements and **100% of functions**, covering
    routing, the API client, both page components, and all five UI components.
  - Frontend function coverage was 79% while statements were at 99%. The gap
    was inline JSX handlers that delegate to a covered helper — the logic was
    tested, but the field name each handler passes in was not, so a mistyped
    key would drop a value on save with nothing to catch it. Parametrised
    wiring tests now walk every form field and sortable header. Worth keeping
    at 100%: it is the only check on those string literals.
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
- **[built] Schema migrations via Alembic** (KAN-15, KAN-16). A column change
  ships as a revision rather than as manual SQL or a dropped table. The
  baseline revision matches the models exactly, verified by autogenerate
  producing an empty diff against a database built from it.
  - `alembic.ini` carries no URL. `env.py` reads it from the app's own
    `Settings`, so connection details have one source of truth, no password
    sits in a committed file, and Alembic honours the same `DATABASE_URL`
    override the tests use.
  - **The test suite migrates rather than calling `create_all`.** `create_all`
    would be slightly faster, but it builds the schema from the models, so it
    would pass whether or not the migrations work — leaving the mechanism that
    runs in production untested. Teardown runs `downgrade base`, exercising the
    downgrade path too. A broken revision now fails the suite, which is the
    point.
  - The baseline was drafted against SQLite, because no server database existed
    when it was written. Autogenerate renders against whichever dialect it
    connected to, so it leaked SQLite spellings — `func.now()` came out as the
    literal `(CURRENT_TIMESTAMP)`, and index creation used the batch form that
    only exists to work around SQLite's inability to `ALTER` in place. Both were
    corrected by hand.
  - **That correction is now verified, not assumed.** The baseline applied
    cleanly to MariaDB 10.11 on the real server (KAN-22): all five indexes
    present, the foreign key cascading, `created_at`/`updated_at` rendering as
    `current_timestamp()`, and `alembic revision --autogenerate` against the
    result producing an empty migration. The models and the deployed schema
    agree on the dialect that actually runs.
- **[decided] Backup is required.** The stated concern is a hard drive crash, so
  the backup must survive the loss of the machine — a local dump on the same
  disk does not satisfy this. Losing the data means losing the job search
  history, which is not reconstructable from any other source.
  - **[decided] Off-site, nightly** (KAN-17).

    **Destination: off-site object storage.** A second copy elsewhere in the
    house would satisfy the stated concern — a failed disk — but not fire,
    theft, or flood, where both copies are lost together. Off-site is the only
    option where no single event takes everything. The cost argument that
    usually pushes against it does not apply here: the database is two tables
    whose bulk is `job_description` and `notes`, so even a long search with
    postings pasted in is single-digit megabytes gzipped. Storage is
    effectively free at this size, which removes the trade-off.

    The specific provider is deliberately left to KAN-18, where credentials
    actually get configured. Whatever is chosen must support scripted upload
    without an interactive login, and per-object deletion so retention can be
    enforced.

    **Frequency: nightly.** Worst case is losing a day — a handful of entries
    that are usually still reconstructable from email and browser history.
    Hourly is affordable at this size but produces 24× the artifacts for a
    tracker touched a few times a week; weekly risks several days of real
    re-entry work.

    **Retention: 30 daily, 12 monthly.** At a few MB each that is a few hundred
    MB at worst. The long tail is not about disk failure — a single recent
    backup covers that — but about damage noticed late, where every retained
    copy is already corrupt if the window is too short.

    **Pruning happens in the backup script, after a new upload is confirmed —
    not by a storage lifecycle rule.** This is the subtle one. A lifecycle rule
    deletes on age regardless of whether anything new arrived, so if uploads
    silently break, it keeps deleting until nothing is left, and the failure is
    invisible until a restore is needed. Deleting only after a confirmed
    successful upload makes a broken backup fail safe: artifacts pile up
    instead of evaporating.

    **Dumps are encrypted before upload**, client-side, not merely at rest on
    the provider. At-rest encryption protects against a stolen datacentre disk;
    it does not protect against the provider or a compromised account. The
    contents are not only the user's own data — the `contacts` table holds real
    people's names, emails, and phone numbers, plus salary figures and notes
    about named companies.

    **The encryption key must be stored somewhere other than the server.** A
    key that lives only on the machine being backed up dies with it, and the
    off-site copies become unreadable — a failure mode that looks exactly like
    a working backup right up until the restore. Where the key lives is part of
    KAN-18.
  - A restore must be tested, not assumed. An unverified backup is not a backup
    (KAN-19).
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
