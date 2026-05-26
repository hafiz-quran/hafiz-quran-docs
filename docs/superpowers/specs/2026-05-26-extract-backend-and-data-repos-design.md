# Extract `backend` and `data` into standalone repos

**Date:** 2026-05-26
**Status:** Approved design, pending implementation plan
**Context:** Prerequisite for the Flutter -> React Native migration. Before the
React rebuild, the backend and data pipeline must live in their own repos so
they belong to neither app (per workspace-root `CLAUDE.md`).

## Goal

Extract two subtrees out of the `hq-flutter` repo into two new standalone git
repos with **preserved history**, and push them to GitHub as **private** repos
under the `hafiz-quran` org:

| Source path in `hq-flutter` | New folder | New GitHub repo (private) |
|------------------------------|------------|---------------------------|
| `supabase/`                  | `hq-backend/` | `hafiz-quran/backend` |
| `scripts/` + `data/`         | `hq-data/`    | `hafiz-quran/data`    |

`hq-flutter` is **left completely untouched** -- nothing is removed from it.
The app keeps shipping; deduplication is deferred to a later, separate task.

## Why extraction (not copy)

The intended end state (workspace `CLAUDE.md`) is five independent repos. The
backend and data pipeline currently live *inside* the Flutter repo. This is an
extraction of existing subtrees, not greenfield scaffolding.

History is preserved because blame/history on migrations and the data pipeline
has ongoing value. The chosen tool is `git filter-repo`.

## Approach: `git filter-repo` on throwaway clones

For each target:

1. Clone `hq-flutter` to a throwaway location under `$CLAUDE_JOB_DIR`.
   (`filter-repo` refuses to run except on a fresh clone -- this is the safety
   property that guarantees the source repo is never modified.)
2. Run `git filter-repo` keeping only the relevant path(s).
3. Move the filtered repo into the prepared empty target folder.
4. Secret-scan the new repo's full history (gate -- see below).
5. Create the private GitHub repo via `gh` and push.

### Backend -- `hafiz-quran/backend`

- Filter: keep `supabase/` only, **hoisted to repo root** (drop the `supabase/`
  prefix via filter-repo path rewriting).
- Resulting layout:
  ```
  hq-backend/        (repo root)
    migrations/
    functions/
    CLAUDE.md
  ```
- Cost of hoisting: commit-history paths read `supabase/migrations/x.sql` while
  the file now lives at `migrations/x.sql`. Accepted -- standard for extracted
  repos, worth a clean daily-use layout.

### Data -- `hafiz-quran/data`

- Filter: keep **both** `scripts/` and `data/` (one `filter-repo` invocation,
  two `--path` args). No hoisting -- they are already two top-level siblings.
- Resulting layout:
  ```
  hq-data/           (repo root)
    scripts/         (Python pipeline code: pyproject.toml, uv.lock, cli.py, ...)
    data/            (generated JSON artifacts: chunks/, lessons/, ...)
  ```
- This sibling layout is **required**: the Python code resolves artifact paths
  via `project_root / 'data' / ...` and tests use `../data/chunks`, where
  `project_root` is the parent of `scripts/` (i.e. the repo root). Keeping
  `scripts/` and `data/` as siblings preserves these assumptions unchanged.

## What carries over / what does not

**Carries:** all commits touching the kept paths, with original authors, dates,
and messages. Mixed commits (which also touched e.g. `mobile-app/`) are kept but
trimmed to show only the relevant diff -- an expected cosmetic effect of any
history-preserving split.

**Does NOT carry** (verified: none are tracked in history):
- Secrets: `scripts/.env` (gitignored via `*.env`, never committed).
- Build/junk: `scripts/.venv/`, `scripts/__pycache__/`, `scripts/.coverage`,
  `data/**/*.log`, the dead `scripts/.pytest_cache` symlink.

## Coupling check -- does this break the Flutter app?

**No.** Verified:
- The app ships a **self-contained** copy of pipeline output under
  `mobile-app/assets/` (lessons, dbs, mushaf pages). It does not read the
  workspace `data/` directory at build time.
- The `import '../data/...'` lines in Dart point to `mobile-app/lib/data/`
  (Dart source), **not** the workspace `data/` dir.
- No build-time reference to `../scripts` or `../data` exists in the app.

## Safety gates

1. **Secret scan before push.** After extraction, scan the new repo's full
   history (`git log --all --name-only` + content grep) for `.env`, secret,
   key, credential, `.p8`, `.pem` patterns. Abort the push if anything
   sensitive appears. Verified pre-emptively that no `.env` is tracked under
   `supabase/`, `scripts/`, or `data/` history.
2. **`.gitignore` forward-port.** Confirm each new repo carries a `.gitignore`
   that keeps `.env`/`.venv`/`__pycache__` ignored at the correct level (the
   `scripts/.gitignore` rules must apply under the new root).
3. **Source untouched.** `filter-repo` runs only on throwaway clones; the
   `hq-flutter` working tree and history are never modified.

## Verification (definition of done)

- `hq-backend/` and `hq-data/` exist as git repos with non-empty extracted
  history (`git log` shows the original commits).
- `hafiz-quran/backend` and `hafiz-quran/data` exist on GitHub as **private**
  repos with the pushed history.
- Secret scan passed on both before push (no `.env`/secret in history).
- `hq-flutter` `git status` is unchanged from before the work (clean, no
  modifications to the source repo).
- Data repo layout has `scripts/` and `data/` as root-level siblings; backend
  repo has `migrations/`, `functions/`, `CLAUDE.md` at root.

## Out of scope

- Removing `supabase/`, `scripts/`, or `data/` from `hq-flutter` (deferred).
- The React Native migration itself (this is only its repo-separation
  prerequisite).
- CI/CD wiring on the new repos.
- Adding the new repos as remotes/submodules anywhere.
