# Audit workflow: commands and flags

This document is for anyone who needs to **run** the maintainer-activity audit from the terminal without reading the implementation. It lists **npm scripts**, **environment variables**, **every CLI flag**, defaults, and **when to use** each option.

**Repository root:** run all commands from the repo root (`community/`), after `npm install`.

---

## Prerequisites

| Requirement | Notes |
|-------------|--------|
| **Node.js** | Same major as the project expects (see `package.json` / CI). |
| **Dependencies** | `npm install` once at repo root. |
| **`GITHUB_TOKEN`** | Required for both scripts. Export in the shell: `export GITHUB_TOKEN=ghp_...` |
| **Token scopes** | At minimum **`public_repo`** so the API can read public org repos and search. Add **`read:org`** if you later extend workflows to sync org team membership via API. |

Never commit the token. Use env vars or your shell profile, not checked-in files.

---

## What the two steps do

1. **`audit:fetch`** — Calls GitHub to list non-archived `asyncapi/*` repos and downloads each repo’s `CODEOWNERS` (from common paths). Writes **`docs/audit/raw-data.json`** (plus `--out` if you override).
2. **`audit:run`** — Reads `raw-data.json`, `teams-mapping.yaml`, and rules, then evaluates maintainer/triager **activity rules** via the GitHub API (Search, commits, etc.). Writes a **new folder** under **`docs/audit/runs/<run-id>/`** with `input/` (snapshots) and `output/` (reports).

You normally **fetch** when CODEOWNERS or the repo list might have changed; you **run** when you want a report (and optionally reuse cached activity results to go faster).

---

## npm scripts (shortcuts)

| Script | Runs |
|--------|------|
| `npm run audit:fetch` | `node scripts/audit-fetch-raw.mjs` |
| `npm run audit:run` | `node scripts/audit-rule-engine.mjs` |

Pass engine flags **after** `--` so npm forwards them:

```bash
npm run audit:run -- --max-repos 3 --refresh-raw
```

---

## Step 1: Fetch raw data (`audit-fetch-raw.mjs`)

### Environment

| Variable | Required | Purpose |
|----------|----------|---------|
| `GITHUB_TOKEN` | **Yes** | Authenticates Octokit for org repo listing and file contents. |

### Flags

| Flag | Argument | Default | Meaning |
|------|----------|---------|---------|
| `--out` | file path | `docs/audit/raw-data.json` | Where to write the JSON snapshot (repo list + parsed CODEOWNERS). |

### Examples

```bash
export GITHUB_TOKEN=ghp_...
npm run audit:fetch
```

Custom output path:

```bash
node scripts/audit-fetch-raw.mjs --out /tmp/my-raw-data.json
```

### When to run

- After **CODEOWNERS** or **repo** changes you care about.
- Before an audit if you want the latest org view **without** coupling it to `audit:run` (otherwise use `--refresh-raw` on the engine; see below).

---

## Step 2: Run the rule engine (`audit-rule-engine.mjs`)

### Environment

| Variable | Required | Purpose |
|----------|----------|---------|
| `GITHUB_TOKEN` | **Yes** | Search API, commits, and other endpoints used to evaluate rules. |

### Flags (complete list)

| Flag | Argument | Default | Meaning |
|------|----------|---------|---------|
| `--rules` | file path | `docs/audit/rules/default.yaml` | Rules file: `window_months`, `rules[]` (kinds, enabled), `bots`, `aggregation`. |
| `--raw` | file path | `docs/audit/raw-data.json` | Input snapshot from the fetch step (or your own copy). |
| `--teams` | file path | `docs/audit/teams-mapping.yaml` | Maps `@asyncapi/...` team slugs to **human** GitHub logins (required where CODEOWNERS names a team). |
| `--max-repos` | integer | no limit | Only process the **first N** repos after sorting (use for quick tests; **not** for full-org policy runs). |
| `--activity-cache` | file path | `docs/audit/activity-cache.json` | JSON file storing cached **per-rule** evaluation results to avoid repeat API work. |
| `--activity-cache-mode` | `merge` \| `all` \| `off` | `merge` | How to use that file (see [Activity cache modes](#activity-cache-modes)). |
| `--refresh-raw` | (none) | off | Before reading `--raw`, run **`audit-fetch-raw.mjs`** with `--out` set to the same path as `--raw`, so CODEOWNERS and repo list are refreshed in one command. |

Invalid `--activity-cache-mode` values are ignored; the engine keeps the default `merge`.

### Examples

**Minimal (defaults):**

```bash
export GITHUB_TOKEN=ghp_...
npm run audit:run
```

**Limit repos (fast local test):**

```bash
npm run audit:run -- --max-repos 5
```

**Refresh CODEOWNERS + repo list, then run the audit** (one command):

```bash
npm run audit:run -- --refresh-raw
```

**Custom rules file (experiment with thresholds without touching default):**

```bash
npm run audit:run -- --rules docs/audit/rules/my-experiment.yaml
```

**Separate activity cache file** (e.g. per branch or experiment):

```bash
npm run audit:run -- --activity-cache docs/audit/activity-cache-experiment.json
```

**Force full refetch of all activity rule evaluations** (still updates the cache file for next time):

```bash
npm run audit:run -- --activity-cache-mode all
```

**Disable activity cache entirely** (always live API; slowest; useful for debugging or verifying no cache influence):

```bash
npm run audit:run -- --activity-cache-mode off
```

---

## Activity cache modes

The activity cache stores **results** of evaluating each **enabled rule** for each **person** in each **repo**, keyed by repo, login, **calendar day** derived from `window_months` (`since_day`), and rule **`kind`**. That way, changing only aggregation or rule weights does not always require re-querying GitHub.

| Mode | Reads cache? | Writes cache? | Use when |
|------|----------------|---------------|----------|
| **`merge`** (default) | Yes | Yes, if new keys were computed | Normal work: repeat runs, rule tweaks, or new repos/users only **fill in missing** keys. |
| **`all`** | No (always compute) | Yes | You want a **full refresh** of activity results (e.g. after a long break or to rebuild cache from scratch). |
| **`off`** | No | No | Debugging, or you must not persist or read cached API results. |

**Changing `window_months` in the rules file** changes `since_day`, so cache keys change: **`merge`** will **fetch only** combinations that are not already in the file.

---

## Outputs (where to look)

After `npm run audit:run`:

| Location | Contents |
|----------|----------|
| `docs/audit/runs/<run-id>/input/` | Copies of `raw-data.json`, `teams-mapping.yaml`, `rules.yaml`, and `activity-cache.json` **if it existed** at start. |
| `docs/audit/runs/<run-id>/output/` | `summary.md`, `summary.json`, `full-report.md`, `full-report.json`, `emeritus-candidates.md`. |
| `docs/audit/activity-cache.json` | Updated when the engine computed new entries and cache mode allows writes. |

The `<run-id>` is a UTC timestamp like `20260329T084806Z`.

---

## Choosing a workflow (cheat sheet)

| Goal | Command |
|------|---------|
| First-time or “I need latest CODEOWNERS” | `npm run audit:fetch` then `npm run audit:run`, **or** `npm run audit:run -- --refresh-raw` |
| Quick test without full org | `npm run audit:run -- --max-repos 3` |
| Tuned rules / aggregation only; data unchanged | `npm run audit:run` (default **`merge`** cache) |
| I changed `window_months` | `npm run audit:run` — **`merge`** fetches missing keys only |
| I want all activity checks redone from GitHub | `npm run audit:run -- --activity-cache-mode all` |
| No cache at all | `npm run audit:run -- --activity-cache-mode off` |

---

## Further reading

- **[README.md](README.md)** — Overview, global files table, rate limits.
- **[rules/RULE_TYPES.md](rules/RULE_TYPES.md)** — What each rule `kind` means.
- **[CHANGELOG.md](CHANGELOG.md)** — Engine version and behavior changes.
