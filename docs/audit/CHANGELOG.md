# Audit pipeline changelog

## 1.2.0

- **Usage guide:** [`docs/audit/USAGE.md`](USAGE.md) — full CLI reference, cache modes, and when to use each command.
- **Activity cache** (`docs/audit/activity-cache.json`): stores per-rule results (`pass`, `reasoning`, `evidence`) keyed by repo, login, window date, and rule kind.
- CLI: `--activity-cache-mode merge|all|off`, `--activity-cache <path>`, `--refresh-raw`.
- Merge mode: reuse cache when present; **fetch and append** only missing keys (new repos/users/kinds/`since_day`).

## 1.0.1

- Search API: pace requests (~2.2s) and retry on `403` rate limit.
- Search API: handle `422` (unsearchable / invalid username) without aborting the run.
- Documented limits and 422 behaviour in `README.md`.

## 1.0.0

- Initial `audit:fetch` and `audit:run` scripts.
- Run layout: `runs/<id>/{input,output}`.
- Rule kinds: `commit_activity`, `merged_pr_author`, `pr_review`, `issue_interaction`, `org_wide_activity`, `last_interaction_any`.
