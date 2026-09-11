# Posting Closed Chrome extension — implementation plan

Date: 2026-09-11. Status: **implemented and tested locally; manual deployment
and Chrome acceptance pending**.

The user approved implementation after the planning session and chose manual
deployment. The endpoint and separate extension now exist. This document keeps
the design and implementation sequence as the cross-repo handoff for Claude or
Codex; the behavior below is implemented unless explicitly deferred. No Jira
issue was created and no changes have been committed, pushed, or deployed.

**Resume here:** [manual deployment and acceptance checklist](POSTING_CLOSED_DEPLOYMENT.md).
Local checks passed: 304 backend tests, 44 extension tests, 122 importer tests,
and the real-JavaScript-to-FastAPI HTTP smoke test against temporary SQLite.
Chrome API behavior in the tests is mocked; actual toolbar and site redirects
still require acceptance checks.

## Goal and scope

The owner opens an existing posting from Job Tracker, visually determines that
it is closed, and clicks a separate Chrome extension's toolbar button. The
extension updates the application whose `job_link` matches that tab's URL to
`posting_closed` (displayed as **Posting Closed**) through the Python API.

The normal path takes one click, with no search, form, or second confirmation.
It updates an existing record only. It does not scrape the page or decide
whether the posting is closed. It does not need the import extension's finite
site catalog: any HTTP(S) posting URL already stored in the tracker can match.

Automatic availability checks and batch updates are explicitly deferred.

## Verified starting point (before implementation)

- `posting_closed` already exists in `app/models.py`, migration
  `b3e51f0a7c46`, frontend labels, badges, filters, and history handling. No new
  enum, database migration, or frontend feature is expected for this work.
- `PATCH /applications/{application_id}` already updates status by ID.
  There is no URL lookup/update endpoint. The list endpoint has pagination and
  defaults to active lifecycle records, so downloading its default page would
  miss records, including already-closed postings.
- `crud.update_application` records actual status transitions in the same
  commit. Repeating the same status does not add a history row. Reuse it;
  `tests/test_status_history.py` guards against unrecorded extra write paths.
- `applications.job_link` is a nullable `String(1024)`, not unique. The create
  duplicate check matches company, title, AND link; it does not establish URL
  uniqueness. PATCH and historical data can also leave duplicates.
- URL validation preserves the submitted string. The import extension sometimes
  stores a canonical URL, e.g. LinkedIn `/jobs/view/{id}/`.
- The existing extension uses Manifest V3, plain JavaScript, `activeTab`,
  `scripting`, a popup, and host access to the tracker. Its API client uses
  `{TRACKER_ORIGIN}/api/applications` with a 12-second timeout.
- The documented deployment is LAN-only with no API authentication. nginx
  supplies the `/api` prefix; FastAPI routes themselves start `/applications`.
  These observations are from local code/docs, not a live deployment check.

## Implemented first-version behavior

1. The user pins the new, visually distinct extension button in Chrome.
2. On click, capture the clicked tab's ID and current URL once. Reject a
   missing URL or non-HTTP(S) scheme locally.
3. Show a per-tab working badge and send one request to the configured tracker.
4. On success, show `DONE` and a tooltip such as
   `Posting Closed: #123 — Acme — Engineer`. A repeat shows `Already Posting Closed`.
5. On failure, show `ERR` and a readable tooltip explaining no match, duplicate
   matches, archived record, validation failure, or connection failure.

Use a Manifest V3 service worker with `chrome.action.onClicked` and no
`default_popup`. Chrome does not dispatch that event when an action popup is
configured. Use per-tab badges/titles for feedback; a persistent popup or
results dashboard is unnecessary for this first version. See Chrome's
[action API](https://developer.chrome.com/docs/extensions/reference/api/action).

Request `activeTab` and host access only to the configured tracker origin.
`activeTab` supplies the invoked tab's URL; requests originate in the extension
service worker. No content script, `scripting`, broad `tabs` permission,
job-site host permissions, or new framework is needed. Sources:
[activeTab](https://developer.chrome.com/docs/extensions/develop/concepts/activeTab),
[extension network requests](https://developer.chrome.com/docs/extensions/develop/concepts/network-requests).

Prevent duplicate in-flight work for the same captured URL. Clear stale
feedback on navigation and on the next click; do not put a result onto a tab
that has since navigated to another posting. Handle a tab closing during the
request without cancelling or misreporting the backend update. Use the existing
12-second timeout convention. A timeout can occur after the server commits:
say the result is unknown and allow an explicit retry, rather than claiming
nothing changed. No automatic retry is needed.

## Implemented API contract

```http
PATCH /applications/by-url/status
Content-Type: application/json

{"job_link":"https://www.linkedin.com/jobs/view/123/","status":"posting_closed"}
```

Deployed URL: `{TRACKER_ORIGIN}/api/applications/by-url/status`.

Use a dedicated request schema: required nonblank HTTP(S) `job_link`, maximum
1024 characters, and `status` restricted to the literal `posting_closed`.
Reject unknown fields so callers cannot accidentally submit unrelated edits.
Keep the existing ID-based PATCH unchanged.

Compact success response, HTTP 200:

```json
{
  "id": 123,
  "company": "Acme",
  "role_title": "Engineer",
  "job_link": "https://www.linkedin.com/jobs/view/123/",
  "status": "posting_closed",
  "changed": true
}
```

| Lookup result | Behavior |
|---|---|
| Exactly one unarchived match | Update using `crud.update_application` and return 200 |
| Match already `posting_closed` | Return 200 with `changed: false`; no additional history |
| No match | 404; no write |
| Multiple matches, including archived matches | 409; no write; report matching IDs for manual resolution |
| Exactly one match, archived | 409; no write; identify the archived record |
| Invalid request | 422; no write |

The archived-record refusal is an extension-specific default. It does
not change the existing ID-based editor's behavior. Do not restrict this action
to Interested: the application's existing policy allows any status transition,
and the user may intentionally close a posting tracked in another status.

Only submit a status update to the existing CRUD helper. Preserve dates,
`next_action` (including `Apply`), next-action date, notes, favorites, links,
contacts, and archive state. Status history and ordinary update metadata may
change. Do not silently clear follow-up fields or archive the application.

## URL matching: keep the first version explicit

Match the current browser URL against stored `job_link` exactly. Do not use
company/title matching, substring search, the first database result, or a
generic rule that strips queries. For example, Indeed's `jk` identifies the
job. Paths and query values can be case-sensitive.

Make exactness independent of database collation. A small implementation can
use SQL equality to select candidates, then Python string equality to verify
them before counting matches; a case-insensitive MariaDB comparison must not
be allowed to select a different posting. Test this distinction explicitly.
No index, URL hash column, uniqueness migration, or stored-link rewrite is
needed at the current scale. The resolver locks candidates using
`with_for_update()` to serialize overlapping updates on MariaDB before reusing
the existing status/history commit. SQLite does not exercise those row locks.

**Known limitation:** opening a stored URL does not guarantee that the final
address bar URL is identical. Tracking parameters, trailing-slash changes,
login redirects, or a redirect to a generic careers page can prevent a match.
Return a clear no-match error and leave manual correction in Job Tracker as
the fallback. Do not infer the original posting from a generic destination.

Before calling the feature ready, try representative links opened from the
tracker on the five currently supported import sites. If common real links
fail exact matching, discuss a narrowly scoped follow-up using verified
site/job identifiers or carrying the original tracker link. Do not quietly
expand this plan into redirect tracking or a canonicalization system.

## Implementation sequence

1. **Backend:** add request/response schemas in `app/schemas.py`, a read-only
   URL resolver in `app/crud.py`, and the dedicated route in
   `app/routers/applications.py`. Resolve all matches before writing. Reuse
   the existing update helper so status and history commit together. Place
   named routes before dynamic routes as a local convention.
2. **Backend verification:** test success, repeat requests/history, missing
   and invalid URLs, duplicates, archived cases, non-Interested status,
   preservation of unrelated fields, and exact-match distinctions (case,
   query, fragment, trailing slash). Verify route reachability. Run the backend
   suite using its protected throwaway SQLite configuration; check matching
   semantics with disposable MariaDB data if relying on database collation.
3. **Separate extension:** recommended sibling repository name
   `chrome-extension-job-tracker-close-posting`. Create a small manifest,
   module service worker, API/config module, distinct PNG icons, tests,
   `README.md`, and `CLAUDE.md`. Use existing client conventions without
   importing scraper code or introducing a shared package. Configure one
   tracker origin consistently in the client and manifest; no settings UI
   is needed initially. Use an unpacked installation like the existing tool.
4. **Extension verification:** test tab capture, one request per click,
   in-flight suppression, all API outcomes, timeout/network/non-JSON failures,
   and tab switching/navigation/closing. Manually check actual Chrome badge
   and tooltip feedback and LAN API access. If Chrome presents a local-network
   permission prompt, document the observed setup step. Do not broaden CORS
   or permissions speculatively.
5. **End to end:** use a disposable tracked record and a representative URL;
   confirm the status and exactly one history transition, then repeat the
   click. Refresh Job Tracker to see the change; a page already open does not
   receive push updates and the Active filter may hide the newly closed row.
   Verify the original import extension still creates records unchanged.
6. **Release and handoff:** deploy the backend before installing the extension
   against it. Update shared requirements/architecture to describe what was
   actually built, record checks and remaining limits, and add the new repo to
   `WORKSPACE.md`. Publish shared docs and bump backend/frontend submodules
   in the normal workflow; never edit their detached `docs/` copies.

## Claude continuity and future work

This plan is stored in `job-tracker-docs`, linked from `WORKSPACE.md`,
`REQUIREMENTS.md`, and the backend's `CLAUDE.md`. The workspace-root Claude
stub already imports shared workspace context. A backend-only session should
read the sibling plan while its `docs/` pointer has not yet been updated.
The new extension's `CLAUDE.md` links to the canonical plan and deployment
checklist and records the API contract, URL behavior, config, and test commands.
Keep deployment-specific extension settings out of reusable context prose.

Resume instruction: **Read the deployment checklist and relevant repo context,
inspect current Git/server state, then help the owner finish manual deployment
and Chrome acceptance.** The implementation session tested only disposable
databases and checked live health/OpenAPI read-only. No production data changed.
The new extension is a separate local Git repository on `main`, without a
remote or initial commit. All implementation/context changes remain uncommitted.

Later, automatic checking can classify postings and submit confirmed closures
through a batch API. Design that when requested, likely addressing records by
ID and reusing the status/history behavior. Network failures, login walls, and
rate limits will need an unknown outcome rather than being treated as closure.
Do not build a scheduler, classifier, batch route, or site adapters now.
