# job-tracker-docs

Shared documentation for the Job Tracker project. This is the **single source
of truth** for anything that applies to more than one repo.

| File | Covers |
|---|---|
| [`REQUIREMENTS.md`](REQUIREMENTS.md) | What the app must do. Authoritative where it overlaps with anything else. |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | How the project is put together, deployment, security posture, current state. |

## Consumed by

Both code repos include this repo as a git submodule at `docs/`:

- [job-tracker-backend](https://github.com/nevans-job-tracker/job-tracker-backend)
- [job-tracker-frontend](https://github.com/nevans-job-tracker/job-tracker-frontend)

Repo-specific design notes are **not** here — they live in each repo's own
root `CLAUDE.md`.

## Editing

Edit these files here, in this repo. A submodule checkout inside a code repo
is a detached-HEAD snapshot; editing there is easy to do by accident and easy
to lose.

Publishing a change is two steps:

```bash
# 1. commit here
git commit -am "Update requirements" && git push

# 2. point each consuming repo at the new version
cd ../job-tracker-backend && git submodule update --remote docs \
  && git commit -am "Bump docs" && git push
```

Step 2 can lag — a stale pointer still resolves to a valid version. Bump it
when the consuming repo needs the newer text.

## Cloning a consumer

```bash
git clone --recurse-submodules https://github.com/nevans-job-tracker/job-tracker-backend.git
```

If you already cloned without it: `git submodule update --init`.
