# AsyncAPI org repository audit (maintainer activity)

This directory holds **global** inputs for the audit pipeline and **per-run** results under `runs/`.

**Terminal reference (all npm commands and flags):** [USAGE.md](USAGE.md).

## Quick start

1. **Token:** create a GitHub PAT with `public_repo` (and `read:org` if you later sync team members via API).
2. **Fetch raw data** (all non-archived `asyncapi/*` repos + `CODEOWNERS`):

   ```bash
   export GITHUB_TOKEN=ghp_...
   npm run audit:fetch
   ```

   Writes `raw-data.json`.  
   **Or** refresh raw data automatically when running the engine: `npm run audit:run -- --refresh-raw` (runs the fetch script first, then the audit).

3. **Edit** `teams-mapping.yaml` so `@asyncapi/...` teams from `raw-data.json` map to member logins (required for team-expanded maintainers).

4. **Run the rule engine:**

   ```bash
   npm run audit:run
   ```

   Common options:

   - `--max-repos N` — only process the first *N* repos (sorted by name), for testing.
   - `--activity-cache-mode merge` (**default**) — reuse cached **per-rule GitHub results** from `activity-cache.json` when the cache key matches; **fetch only missing** keys (e.g. new repos, new maintainers, new rule `kind`, or new `since_day` after you change `window_months`).
   - `--activity-cache-mode all` — ignore cache reads; refetch every rule for every person (still **writes** the cache at the end for next time).
   - `--activity-cache-mode off` — do not read or write the activity cache (always live API; slowest).
   - `--activity-cache /path/to/activity-cache.json` — custom cache file (default: `docs/audit/activity-cache.json`).
   - `--refresh-raw` — run `audit:fetch` before loading `raw-data.json`.

Outputs go to `docs/audit/runs/<run-id>/`:

- `input/` — **snapshot** of `raw-data.json`, `teams-mapping.yaml`, rules file, and `activity-cache.json` (if it exists when the run starts).
- `output/` — `full-report.json`, `full-report.md`, `summary.md`, `summary.json`, `emeritus-candidates.md`.

## Global files

| File | Purpose |
|------|---------|
| `raw-data.json` | Repo list + parsed `CODEOWNERS` (maintainers, teams, triagers). Regenerate with `audit:fetch`. |
| `teams-mapping.yaml` | `teams["asyncapi/slug"].members: []` for humans in org teams. |
| `rules/default.yaml` | Rule kinds, `window_months`, bot deny lists, aggregation (`profile` / `k_of_n` / `none`). |
| `rules/RULE_TYPES.md` | Documentation for each `kind` string. |
| `rules/schema.json` | JSON Schema (informal) for the rules file. |
| `emeritus-log.md` | **Human / TSC** log after reviewing `emeritus-candidates.md` (not overwritten by the engine). |
| `activity-cache.json` | Cached **rule evaluation** results keyed by repo, user, `since_day` (from `window_months`), and rule `kind`. Speeds up repeat runs when rules or aggregation change but GitHub data is unchanged. |

Cache keys include **`since_day`**; changing `window_months` in the rules file creates **new** keys, so the first run after a change fetches only those new combinations (merge mode).

## Spec Kit

Product specs and tasks for this pipeline live under [`../../spec/audit`](../../spec/audit). Upstream toolkit: [github/spec-kit](https://github.com/github/spec-kit).

## Comparing runs

Diff two folders:

```bash
diff -u docs/audit/runs/RUN_A/output/summary.md docs/audit/runs/RUN_B/output/summary.md
```

## Rate limits and edge cases

- **Search API:** Authenticated users get about **30 search requests per minute**. The engine **paces** search calls (~2.2s apart) and **retries** after `403` when GitHub reports a secondary limit reset.
- **Full-org runs** can take **15–45+ minutes** depending on maintainer count and network. Use `--max-repos` for quick iteration.
- **422 on search:** Some handles in `CODEOWNERS` are not searchable (placeholders like `global-owner1`, renamed users, or usernames GitHub rejects in `author:`). Those rules **fail** with an explanatory reason instead of crashing the run.
- **Hyphenated usernames** (e.g. `devilkiller-ag`) may return 422 from Search in some cases; fix the handle in `CODEOWNERS` or rely on `commit_activity` when search fails.
