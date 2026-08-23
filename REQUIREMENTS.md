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
| `company_size` | enum | no | Wellfound's six bands — see below |
| `years_experience_min` | smallint | no | Minimum only; `0` means entry level and is distinct from blank |
| `status` | enum | yes, defaults `applied` | Eight values, freely assignable — §3 |
| `salary_min` | decimal(10,2) | no | Must not exceed `salary_max` |
| `salary_max` | decimal(10,2) | no | |
| `salary_currency` | string(10) | no, defaults `USD` | |
| `date_applied` | date | no | Empty means not applied for yet, and pairs with status `interested` — §3. A future date warns rather than rejects |
| `notes` | text | no | |
| `next_action` | string(255) | no | What is owed next ("follow up", "take-home due") |
| `next_action_date` | date | no | When it is owed; gives the list an actionable sort |
| `job_description` | text | no | Snapshot of the posting, which outlives the link |
| `cover_letter` | text | no | What was written to this employer — see below |
| `archived_at` | datetime | no | Archive marker, indexed; `NULL` means active — §4.1 |
| `created_at` / `updated_at` | datetime | auto | Server-side defaults |

**[decided] Company size uses Wellfound's bands** (KAN-35), adopted rather
than invented so the values match what the postings already say:

| Stored value | Label |
|---|---|
| `seed` | Seed (1–10 employees) |
| `early` | Early (11–50 employees) |
| `mid_size` | Mid-size (51–200 employees) |
| `large` | Large (201–500 employees) |
| `very_large` | Very Large (501–1000 employees) |
| `massive` | Massive (1001+ employees) |

An enum here and free text for `source` is not an inconsistency. `source`
stayed free text because its real values were not yet known; these are known,
closed, externally defined, and bounded by employee count, so any company maps
to exactly one band.

- **The cost is that they are Wellfound's taxonomy, not ours.** If they change
  it, or exact headcount is wanted later, that is a migration rather than an
  edit. Recorded so the choice is revisitable rather than mysterious.
- **The labels carry the ranges, and that is not decoration.** "Large" means
  nothing without "201–500", and choosing correctly is the entire point of a
  controlled list. One exported `COMPANY_SIZE_LABELS` map is the only place a
  band is spelled for a human — the KAN-34 pattern, applied from the start
  rather than after the duplication has to be cleaned up.
- **Declared smallest to largest**, which is load-bearing rather than tidy:
  MariaDB stores an `ENUM` as its ordinal, so this order is what makes sorting
  by the column mean band order.

**[decided] `years_experience_min` stores the minimum as an integer** (KAN-32).
"3–5 years" and "5+" both store as `3` and `5`. Chosen over free text so the
column can be sorted and filtered — *"show everything wanting under five
years"* is a query worth having — at the accepted cost that the posting's exact
wording is lost and "senior level" has to be translated by hand.

- Negative values are rejected; nobody means to enter one. No upper bound is
  imposed, because there is no equally obvious line — 30 is unusual but real.
- **`0` is a real answer and distinct from blank.** An entry-level posting
  states no minimum, which is not the same as not stating one.

**[decided] The cover letter is stored as text, not as a file** (KAN-40).
§6.2 deferred *file attachments*; the requirement turned out to be narrower —
the text is what is wanted, and a PDF can be regenerated from it. That
distinction is worth almost all of the work:

- **The restore rehearsal keeps meaning what it means.** KAN-19 and KAN-37
  verify the *database* restores. A second store on the filesystem would have
  left a green `RESTORE VERIFIED` sitting beside a bucket missing every cover
  letter — the false-confidence failure this project keeps designing against.
- **No new encryption path.** §5 requires client-side encryption before
  upload, and a cover letter carries the owner's name and history. Syncing
  files would have needed `rclone crypt` built first.
- **Nothing can diverge**: no row pointing at a missing file, no orphan file.
- It is searchable in a way a PDF never would be — though not yet, see the
  gap in §4.2.

Measured, for a one-page letter: **~1.3 KB as text, ~1.6 KB as HTML, 40–80 KB
as a PDF.** The detail screen offers a **Download as HTML** control producing a
small document — paragraphs and typography intact, opens in Word or Google
Docs, ready to export as the PDF that gets attached.

**[built] A `.docx` can be uploaded and converted** (KAN-41), so a letter
written in Word keeps its bold, lists and headings instead of being retyped.

- **The conversion happens in the browser**, lazy-loaded, and the file
  *structurally never reaches the server* — no upload endpoint, no multipart,
  no CPU on a single core, nothing to clean up. The converted HTML goes out
  through the ordinary PATCH.
- **`.docx` only, deliberately.** A `.docx` is structured XML where paragraphs
  and bold runs exist as data. A PDF is positioned glyphs with no structure:
  paragraph breaks would be guessed, bold would be gone, a letterhead would
  land in the body as prose. A PDF is also a *rendering of the `.docx` you
  already have*. If only a PDF exists, open it and paste — the viewer already
  does that extraction, for no bytes and no dependency.
- **The column now holds either prose or HTML**, and one function reconciles
  them on read. No format flag on the row and no migration: `Text` holds both,
  and anything not recognised as our HTML is escaped as text — wrong-looking at
  worst, never unsafe.
- **An allowlist sanitiser runs on the way in and on the way out.**
  "Constrained by the converter" is not "sanitised", and the column is still
  writable through the API. `IMG` is deliberately excluded, and that is
  load-bearing: the converter inlines embedded images as base64 data URIs, so a
  letterhead logo would turn a 2 KB column into 100 KB+ and then sit in every
  nightly backup forever.
- **Cost:** one runtime dependency, +26 packages. Lazy-loaded, so the initial
  bundle is unchanged at ~207 KB and the converter's 130 KB gzipped is fetched
  only by someone who actually uploads.

**[decided] `date_applied` is optional** (KAN-31). The tracker has to hold a
job you intend to apply for, not only ones already sent — that is the half of
a job search where the decisions are still open. An empty date is not a
missing value to be filled in later by default; it is a statement that the
application has not gone out, and §3's `interested` status is what says so.

**This table is the baseline revision plus one** (§5), and it is
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
  - **[built] `include_contacts` opts back in** (KAN-39), for the CSV export
    and nothing else. It is a parameter rather than a change of default, and
    it eager-loads with `selectinload`, so it costs one extra query for the
    whole page rather than one per row — the cost the rule above exists to
    avoid. The list screen never sets it.

### 2.2 Status history

**[built] Every status change is recorded** (KAN-42), in a `status_changes`
table related many-to-one to applications.

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | int | auto | |
| `application_id` | int, FK → `applications.id` | yes | Cascades; indexed with `changed_at` |
| `from_status` | enum | no | `NULL` opens an application's history rather than marking a transition |
| `to_status` | enum | yes | Indexed with `changed_at` |
| `changed_at` | datetime | auto | |

**Nothing reads it yet, and that is the point.** History cannot be
reconstructed after the fact — the applications table holds only the current
status, and `updated_at` says when a row last changed rather than what it
changed from. So the recording shipped on its own, ahead of any timeline or
graph, because every day without it is a day permanently missing from whatever
gets built on it later.

- **`changed_at` is when the *record* was edited, not when the thing
  happened.** A rejection email left unread for a week charges that week to the
  previous status. Fine for "three weeks in Applied"; potentially most of the
  value for "five days in phone screen". **The fix, deferred with a trigger:**
  a second `effective_at` column, defaulting to `changed_at` and correctable by
  hand — to be added the first time a recorded duration is visibly wrong enough
  to matter.
- **`date_applied` does not have this problem**, being a real-world date the
  user types. A graph of *applications per day* should therefore read
  `date_applied`, not this table. This table is what makes every *other* status
  answerable.
- **Only a real move is recorded.** The detail screen sends every field on
  every save, so the status arrives unchanged constantly; recording those would
  bury the transitions and make every computed duration read as zero.
- **The recording is a convention, not an assertion.** It is called explicitly
  from `crud`, so it fires only where someone remembered to call it. A test
  parses `crud.py` and fails if a third function starts assigning attributes,
  because history going silently incomplete is the worst failure mode — nothing
  looks wrong until a timeline months later turns out to have holes.
- **The five pre-existing applications got one row each at the migration
  timestamp**, with a `NULL` from_status. We knew each one's current status but
  not how it got there, so a row claiming `created_at → current status` would
  have been fiction. Stamping "as of now, it is this" is true and invents
  nothing — at the cost that for those five, history begins at the migration
  rather than at creation.
- Its downgrade refuses while any row exists, and the reason is stronger than
  for the revisions before it: those dropped columns whose values could be
  retyped from a posting, whereas this is the only copy of when each status
  changed.

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

Eight statuses: `interested`, `applied`, `phone_screen`, `interview`, `offer`,
`rejected`, `ghosted`, `withdrawn`.

**[decided] `interested` marks a job not yet applied for** (KAN-31). It is
the pair to an empty `date_applied` (§2): without it, a job you intend to
apply for sits labelled **Applied** with nothing to say when, and the status
filter cannot separate intentions from applications.

- **A create with no date and no stated status is stored as `interested`**,
  not `applied`. Only an *absent* status is reinterpreted — an explicit
  `applied` with no date is honoured, because the free-assignment rule below
  means the user is allowed to say odd things deliberately.
- The new-entry form mirrors this rather than duplicating it: while the user
  has not picked a status, clearing the date shows **Interested** and entering
  one shows **Applied**. The form always sends a status, so without this the
  select would read Applied while the record being created is not.
- **In the database it is appended to the enum, not placed first.** MariaDB
  stores an `ENUM` as its ordinal, so appending is the only change that leaves
  existing rows meaning what they meant. Display order is the frontend's, where
  `STATUS_LABELS` lists it first; the consequence is that sorting the list *by
  status* on MariaDB puts Interested last.

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
  monotonic progression through statuses. §2.2 now records every transition,
  including backwards ones, so a timeline reading it must handle
  `rejected → interview` as ordinary.

---

## 4. Functional requirements

### 4.1 Managing applications

- **[built]** Create an application via a form; company and role title are
  required, everything else — including date applied — optional.
- **[built]** Edit any field of an existing application.
- **Superseded** — the former delete action is gone, replaced by Archive below.
  There is no DELETE route for applications and no delete call in the API
  client; the route responds 405.
- **[built]** New applications default to today's date and status `applied`.
  Clearing the date switches the status to `interested` until the user picks
  one themselves — see §3.
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

  **[decided] A NULL sorts as though it were greater than every real value**
  (KAN-31). Once `date_applied` became optional this stopped being an
  implementation detail: the default sort is date applied **descending**, and
  both MariaDB and SQLite put NULLs last in that direction — so the jobs not
  yet applied to would sink to the bottom, which past 50 rows means below the
  Load more control (§4.3) and off-screen entirely.

  Treating NULL as the largest value fixes that with a rule rather than a
  special case. It is also the honest reading: an application not yet sent has
  no date because that date, if it ever exists, is in the future. So
  descending puts un-applied jobs at the top where the action is, ascending
  puts them at the end, and reversing the sort still reverses the whole list —
  nothing is pinned.

  - Applied to **every** sortable column, not just dates, so there is one
    rule. Consequence: sorting ascending now puts an empty Location, Source
    or Next action last, which is the conventional expectation and was
    previously the other way round.
  - Implemented as a leading `IS NULL` sort key, because MariaDB has no
    `NULLS FIRST` / `NULLS LAST`. The comparison yields 0 or 1 on both it and
    SQLite, so the tests and the deployment agree.

  **Enum columns sort differently on the two dialects, and the tests cannot
  see it.** MariaDB stores an `ENUM` as its ordinal, so `status` and
  `company_size` sort in declaration order there. SQLite has no enum type and
  stores the same columns as `VARCHAR`, so they sort alphabetically — and
  SQLite is what the test suite runs on. This predates KAN-35; it has been
  true of `status` since the baseline and was simply never written down.

  - It only affects **ordering**, never which rows come back or what they
    contain, so it is a display-order wart rather than a correctness problem.
  - Declaration order for both enums is the meaningful order — lifecycle for
    `status`, smallest-to-largest for `company_size` — so the *deployment*
    behaves correctly. The divergence is that a test asserting band order
    would pass or fail for reasons unrelated to the deployment, which is why
    none does.

  The API accepts a wider set than the table exposes — `salary_min`,
  `salary_max`, and `created_at` are permitted too — and rejects anything else
  with a 422. That whitelist is a security boundary, not a convenience:
  `crud.list_applications` resolves the column with `getattr`, so the pattern
  on the route is the only thing preventing an arbitrary attribute lookup.
- **[decided] Salaries are displayed in thousands, and USD is not labelled**
  (KAN-36). `106,400–177,300 USD` becomes `106K–177K`. This is list display
  only — the detail screen keeps raw numbers in its editable fields, and the
  stored `decimal(10,2)` is untouched, so nothing is lost.

  - **The suffix is dropped only when the currency *is* USD**, not always.
    Every job in this search pays in USD, so the label is noise on every row of
    a column that only survives on wider screens. Dropping it unconditionally
    would show a misleading bare number for a non-USD entry, so
    `salary_currency` keeps its meaning and a `GBP` row still says `GBP`.
  - **Values below 1000 are shown unrounded.** An hourly rate entered as `55`
    would otherwise render as `0K` — not merely ugly but wrong. Decided rather
    than discovered.
  - Rounding is to nearest, so `106,500` reads `107K`. Truncating would
    understate the figure.
  - **[built] The currency is no longer editable from the form** (KAN-38). It
    was a free-text input, which is how `A$` reached a `Remote (United States)`
    role and then the list. Given the premise above — every job in this search
    pays in USD — the input could not record anything useful, only something
    wrong. A create now takes the API's `USD` default and an edit leaves the
    stored value alone.

    This is not the free-text-versus-enum argument from `source` and
    `company_size`: a controlled list would be no better, because the problem
    is not that the values fragment but that there is nothing to record.

    **The display suffix stays**, and the reasoning runs opposite to the
    obvious one. It looks like dead code now that the form cannot produce a
    non-USD value — but that suffix is the only reason the bad data was
    noticed. Dropped unconditionally, `202K–210K` would have looked perfectly
    normal and the wrong currency would have sat there indefinitely. The column
    is still settable through the API, so the label keeps a stray value visible
    rather than letting it read as dollars.

    `salary_currency` stays in the schema. Dropping it is a migration for no
    gain, and if non-USD work ever matters the field returns as a controlled
    list rather than free text.

- **[built] The filtered list exports to CSV** (KAN-39). A green **Export
  CSV** control sits left of **+ Add application**, producing a file that
  opens in Excel. CSV rather than `.xlsx` because it needs no Excel-specific
  library on either side.

  - **Every field, plus contacts** — not the table's columns. The table hides
    most fields because of screen width, which is not a constraint a
    spreadsheet has.
  - **Values are written for calculating, not for reading.** `salary_min` and
    `salary_max` are two numeric columns rather than the list's `106K–177K`,
    which Excel would treat as text; dates stay ISO so Excel parses them;
    `status` and `company_size` carry their readable labels. Timestamps have
    their `T` replaced with a space, which is the difference between Excel
    seeing a date-time and seeing a string.
  - **Every row the filters match, not the page on screen.** The list
    paginates at 50 (§4.3); the filter is the intent and the page size is an
    artifact of scrolling. Handing over 50 of 120 rows without saying so is
    the same silent truncation §4.3 exists to have fixed.
  - **One row per application, contacts flattened into numbered columns**
    (`Contact 1 name`, …), widened to the busiest exported application. A row
    per *contact* would repeat the application, so any sum over salary would
    multiply-count it and the row count would stop meaning "applications".
    Two files joined on `application_id` is the properly relational answer and
    is where to go if applications ever routinely carry many contacts.
  - **The file is built in the frontend**, not the API. `STATUS_LABELS` and
    `COMPANY_SIZE_LABELS` live there, and a server-side exporter would need a
    second copy of both in Python — the duplication KAN-34 existed to remove.
  - `cover_letter` is included, on the same "every field" rule, but **with its
    markup stripped back to prose** — a spreadsheet cell full of `<p>` tags is
    noise, and this file's rule is values written to be worked with. Note it
    compounds the point below: the file already leans on `job_description`,
    and this is a second long-form column. If the spreadsheet becomes
    unwieldy they are the two to drop, and they should go together.
  - **Three things Excel would otherwise get wrong**, each guarded and tested:
    a cell starting `=`, `+`, `-` or `@` is *executed* as a formula, and notes
    are pasted from postings, so string fields are prefixed with an apostrophe;
    the file opens with a UTF-8 BOM, without which Excel on Windows mangles
    every en-dash and the company-size labels are full of them; and fields are
    quoted per RFC 4180 with embedded quotes doubled, because job descriptions
    carry commas, quotes and newlines.
  - Disabled when the filters match nothing — a headers-only file is a puzzle
    rather than a deliverable.

- **[built]** Free-text search across company, role title, and location,
  debounced 250ms.
- **[built]** Filter to a single status, or all statuses.
- **[built]** Result count displayed; empty state when no applications exist.
- **[gap]** Search does not cover `notes`, `source`, `job_description` or
  `cover_letter`. Notes are where the useful detail lives (recruiter names,
  interview feedback), and *"which letter did I say that in?"* is a natural
  question the tracker cannot answer. The case for closing this grows with
  every long-text field added — KAN-40 made it four.
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
- **[built] An "+ Add application" control sits on the detail screen** as well
  as the list (KAN-33). Saving a new entry lands on its detail screen, so
  without this every subsequent entry meant a trip back to the list — save,
  add, save, add is the flow this exists for.

  - **It is deliberately absent on the new-entry screen.** The two screens are
    the same component, so it would have come "free" — but it would navigate
    to the route already showing. There is nothing to add to while you are
    already adding, and a control that does nothing is worse than no control.
  - **Both routes render that one component, so React reuses the instance
    rather than remounting.** Going from a record to `/applications/new` must
    therefore clear the loaded record explicitly, or the new screen comes up
    carrying its values and one Create makes a duplicate. This is a property
    of the shared-component decision in this section, not an implementation
    detail — anything else added to that screen inherits it.

- **[built] Leaving the detail screen with unsaved edits now warns.** The
  detail screen *is* the edit form, so navigating away discards typing with no
  trace and retyping is the only recovery. That is the irreversible case §4.1
  reserves confirmations for — unlike archive, which is one click to undo and
  therefore prompts for nothing.

  - It guards **both** exits, the back link and the add button. The back link
    had the hazard first; guarding only the newer one would have been
    arbitrary.
  - Cancel and Save are not guarded. Cancel *is* the discard.
  - It only interrupts when something would genuinely be lost, so an untouched
    record navigates away silently.

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
  - **Coverage as measured:** backend 157 tests, 99% of statements — the only
    uncovered line is the MySQL URL branch, which tests never take by design.
    Frontend 286 tests, 99% of statements and **100% of functions**, covering
    routing, the API client, both page components, and all five UI components.
  - Frontend function coverage was 79% while statements were at 99%. The gap
    was inline JSX handlers that delegate to a covered helper — the logic was
    tested, but the field name each handler passes in was not, so a mistyped
    key would drop a value on save with nothing to catch it. Parametrised
    wiring tests now walk every form field and sortable header. Worth keeping
    at 100%: it is the only check on those string literals.
  - **Coverage says nothing about layout, and this bit for real.** jsdom does
    not lay out — it has no viewport, no widths, no overflow. KAN-25 found the
    new-application form unusable in portrait on a phone, with the required
    Company and Role title fields pushed off-screen, while all 137 tests passed
    at 100% function coverage. No unit test in this stack could have caught it.
    Responsive behaviour is verifiable only against a real viewport, which is
    why §1 treats a session on the actual device as a requirement rather than a
    nicety.
  - **[built] The runs are automated** (KAN-26). A systemd timer fires
    `deploy/run-tests.sh` nightly at 03:00 with `Persistent=true`, so a machine
    that was off overnight runs once after boot rather than silently skipping.
    The same script is what to run after a deploy.
    - **A failure has to be visible, so it is reported at SSH login.** There is
      no MTA on the machine, and a nightly suite whose failures land in a log
      nobody reads produces false confidence rather than none. The script
      writes a status file and an `update-motd.d` hook prints it — one quiet
      line on success, red with both exit codes on failure.
  - **The test suite destroys data, so it must never reach the live database.**
    `tests/conftest.py` empties every table after each test. On the server this
    margin is thinner than it looks: the backend's `.env` sits in the working
    directory holding live credentials, and `Settings` is built once at import
    time — so anything that touched `app.config` before conftest set
    `DATABASE_URL` would build a production engine and the suite would truncate
    the job search history.
    - **The protection is an assertion, not a convention.** conftest checks the
      engine it actually received and refuses to run unless it is SQLite. That
      check is on the engine rather than the environment, so it holds no matter
      who runs pytest, from which directory, carrying which variables.
    - Verified by reproducing the hazard rather than reasoning about it:
      importing `app.config` first yields a MySQL engine, and pytest then
      aborts at conftest load. No tests run and no connection is attempted.
    - `run-tests.sh` also exports its own `DATABASE_URL` and never sources the
      service's `.env`. Note this is belt-and-braces only — conftest sets
      `DATABASE_URL` unconditionally and therefore overrides it. The engine
      check is what actually protects the database.
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
  - **[built] The first revision that alters an existing table** shipped with
    KAN-31 — `date_applied` made nullable and `interested` appended to the
    status enum. Two things about it are deliberate:
    - It uses `batch_alter_table` for both changes. SQLite cannot ALTER a
      column in place and the tests run on SQLite; on MariaDB batch mode
      emits a plain ALTER. One code path rather than a dialect branch.
    - **The downgrade refuses rather than guesses.** Reversing it is lossy —
      there is no honest date for a record that never had one. It counts the
      rows that depend on what the revision added and raises if any exist, so
      a downgrade either reverses cleanly or does nothing. Left to MySQL
      outside strict mode, restoring NOT NULL over a NULL writes `0000-00-00`
      without complaint.
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
  - **[built] A restore has been verified** (KAN-19), not assumed — an
    unverified backup is not a backup.

    Rehearsed against an artifact the timer produced, fetched from the
    provider, and decrypted with the passphrase held in the password manager
    rather than the copy on the server. That distinction is the substance of
    the test: it is the only check that catches a wrong off-server key, and it
    nearly failed for real when the password-manager entry did not save during
    KAN-18, leaving the passphrase only on the machine being backed up.

    Restored into a scratch database and compared against the live one: row
    counts, the Alembic revision, and read-backs of the archived record and the
    contact join — so `archived_at` and the foreign key are exercised rather
    than only the table shapes. **42 seconds**, procedure recorded in
    `WORKSPACE.md`.

    **Repeated after the first schema changes shipped** (KAN-30), which is the
    rule below being followed rather than merely stated: `4500fe76cbd9` and
    `127a196f3c90` had both landed, so the artifact carried a schema the
    original rehearsal never restored into. **25 seconds**, every comparison
    matching, including the Alembic revision on both sides.

    **Repeated a third time after KAN-40** shipped `53f76402812f` — **15
    seconds**, every comparison matching. That one is different in kind: it
    happened because the machine asked for it. Nobody remembered the rule; the
    MOTD hook below noticed the migration and said so at login.

    Repeat after any schema change ships through Alembic. A restore path is
    only as good as the schema it restores into.

    - **[built] The rehearsal records itself** (KAN-37). It writes a status
      file carrying the revision it verified against, and an
      `update-motd.d` hook reports it at SSH login — so the repeat rule is
      enforced by the machine rather than by remembering. The third such
      hook, for the same reason as the other two: an unread result is false
      confidence rather than none.
      - **The signal is drift, not age**, unlike the backup hook. Weeks
        between rehearsals is expected; a migration shipping is what makes
        one overdue. The hook compares the recorded revision against the
        newest in `alembic/versions`, read off disk so login stays instant
        and needs no credentials.
      - A `MISMATCH` persists until a passing run replaces it.
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
| File attachments — the *file itself* | Needing the exact artifact that was sent. The text half is built (KAN-40), so this is now only about fidelity to the original document |
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
