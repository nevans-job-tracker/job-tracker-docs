# Posting Closed extension — manual deployment

Prepared 2026-09-11. The owner selected **manual deployment**.
**Backend deployment confirmed from the owner's terminal screenshot:**
`develop` at `16398b1`, service active/running, startup migration successful,
304 server tests passed (99% statements), health OK, and the closure endpoint
present in OpenAPI. The owner subsequently tested the extension manually and
reported that it worked. **The initial manual closure feature is complete.**
The steps below remain as the installation/verification reference for future use.
Update after public-repository preparation: the extension now has a sanitized
initial commit. Its `main` was subsequently published and verified at `3ba772c`
on 2026-09-11. Its configured `manifest.json` is ignored and preserved
locally; only `manifest.example.json` is tracked. Do not force-add the local
manifest. Verify current Git/server state before repeating earlier steps.

## 1. Publish the local changes from Windows

The workspace is `C:\Users\evans\Projects\job-tracker`. Run these commands in
PowerShell, stopping on any failed command. Review `git status` before each
commit; the file lists below cover this feature and its planning/context work.
Backend branch should be `develop`, docs branch `main`.

```powershell
Set-Location C:\Users\evans\Projects\job-tracker
git -C job-tracker-docs status --short
git -C job-tracker-docs add README.md WORKSPACE.md REQUIREMENTS.md ARCHITECTURE.md POSTING_CLOSED_EXTENSION_PLAN.md POSTING_CLOSED_DEPLOYMENT.md
git -C job-tracker-docs commit -m "Document Posting Closed extension and manual deployment"
git -C job-tracker-docs push origin main

git -C job-tracker-backend submodule update --remote docs
git -C job-tracker-backend status --short
git -C job-tracker-backend add app/schemas.py app/crud.py app/routers/applications.py tests/test_status_by_url.py CLAUDE.md README.md docs
git -C job-tracker-backend commit -m "Close an existing posting by exact URL"
git -C job-tracker-backend push origin develop

git -C chrome-extension-job-tracker-close-posting status --short
# The extension's initial commit already exists. Never add the ignored manifest.json.
```

The extension is already published on `main` at
`https://github.com/nevans-job-tracker/chrome-extension-job-tracker-close-posting`;
`origin/main` tracks it. Skip initial commit/remote setup for that repository.
It can be loaded directly from this folder. No frontend build is needed. Its
docs submodule can be bumped later; the workspace-root Claude context already
reads the up-to-date sibling docs directly.

If a push is rejected because the remote advanced, stop and reconcile it;
do not force-push. There is no need to fast-forward `main` until deployment
has passed the project's normal checks.

## 2. Deploy the backend on the server

Connect using your usual SSH method, then:

```bash
cd /opt/job-tracker-backend
git status --short
git pull --ff-only origin develop
git submodule update --init
sudo systemctl restart job-tracker-backend
sudo systemctl --no-pager --full status job-tracker-backend
.venv/bin/python -m pytest
curl -fsS http://localhost/api/health
curl -fsS http://localhost/api/openapi.json | .venv/bin/python -c 'import json,sys; assert "patch" in json.load(sys.stdin)["paths"]["/applications/by-url/status"]; print("Closure endpoint is present")'
```

If the initial status shows local modifications, inspect them before pulling.
No dependencies or schema migrations were added by this feature. The service's
normal startup migration step still runs. The test suite is guarded to use
throwaway SQLite; never remove that guard.

Read-only server check during implementation: health was OK on `develop` at
`771e411`, and the closure route was absent. Local backend HEAD was `d26fa89`,
which also contains the already-committed KAN-76 insights archive-exclusion fix
and its docs bump. Pulling current `develop` brings those commits as well.

## 3. Install the extension in Chrome

1. Open `chrome://extensions` and turn on **Developer mode** if needed.
2. Select **Load unpacked**, then choose:
   `C:\Users\evans\Projects\job-tracker\chrome-extension-job-tracker-close-posting`.
3. Pin **Job Tracker — Posting Closed** from the Extensions menu. Its icon is
   an orange briefcase with a minus; the original import extension is separate.

The existing local manifest remains configured for the current LAN tracker.
A fresh clone must create `manifest.json` from `manifest.example.json` and set
the tracker origin, either manually or with `npm run configure -- <origin>`.
The local manifest is intentionally ignored by Git. No build is needed.
If Chrome asks for a local-network permission, review/handle that prompt
yourself and report the exact wording if access fails.

## 4. Verify the actual browser workflow

- Choose one existing, unarchived posting that you have visually confirmed is
  closed. Open its saved link from Job Tracker and wait for loading to finish.
- Click the new extension once. Expect `DONE`; hover to check that its tooltip
  names the intended record. Refresh Job Tracker and confirm Posting Closed
  plus one status-history transition. Active filters may hide the changed row.
- Click again on the same page. Expect `Already Posting Closed` and no extra
  history entry. Notes, next action, dates, and favorites should remain intact.
- Navigate away: the prior badge/tooltip should clear. An untracked page should
  produce `ERR` with a no-match explanation.
- Try actual links opened from the tracker on Wellfound, LinkedIn, Indeed,
  Built In, and Dice as applicable. Only click for jobs you intend to mark
  closed. If the site redirects and exact matching fails, retain the original
  and final URLs for diagnosis; use the tracker editor as the current fallback.

**Local verification already passed:** 304 backend tests (99% statements),
46 extension tests, 122 existing importer tests, and the real-HTTP smoke test
using five representative URL forms and a temporary migrated SQLite database.
Chrome API behavior is mocked in the automated tests. The owner confirmed a
successful manual Chrome test on 2026-09-11. The report did not specify the
posting/site, repeat-click history checks, or concurrency behavior; do not
claim exhaustive live coverage of redirects or MariaDB row locks.

**No further user action is required for the first version.** Use the extension
normally. If a URL fails to match, collect the stored and final URLs for
diagnosis. Automated availability checks and batch updates remain deferred.
