# Deploy and CI

CI on every PR, the environments, the first deploy, remote migrations with backups, secrets, rollback, smoke tests and monitoring. **The owner approves every deploy, remote migration, secret change and production data change.** Agents prepare the commands and the checklist; `devops` runs them only with the go-ahead.

## Contents
1. [CI with GitHub Actions](#1-ci-with-github-actions)
2. [Environments](#2-environments)
3. [First deploy checklist](#3-first-deploy-checklist)
4. [Every release](#4-every-release)
5. [Remote migrations with a backup](#5-remote-migrations-with-a-backup)
6. [Secrets](#6-secrets)
7. [Rollback](#7-rollback)
8. [Smoke tests](#8-smoke-tests)
9. [Monitoring and logs](#9-monitoring-and-logs)
10. [Known problems](#10-known-problems)

## 1. CI with GitHub Actions

`.github/workflows/ci.yml` runs on every PR and on pushes to `main`: install with the lockfile, typecheck, lint, React Doctor (`pnpm doctor`, fails on errors), test (Worker tests in workerd), build.

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  check:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4        # reads "packageManager" from package.json if set
        with:
          version: 10
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm doctor
      - run: pnpm test
      - run: pnpm build
```

- `worker/worker-configuration.d.ts` is committed (regenerated with `pnpm cf-typegen` after `wrangler.jsonc` changes), so CI doesn't need Cloudflare credentials.
- Branch protection on `main` (owner's choice): require the `check` job and one review; no direct pushes.
- Optional, when the owner wants it: a deploy job on `main` after `check`, using a `CLOUDFLARE_API_TOKEN` repository secret (scoped to "Edit Cloudflare Workers" on one account) and `CLOUDFLARE_ACCOUNT_ID`:

```yaml
  deploy:
    needs: check
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production               # GitHub environment with required reviewers = owner approval
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 10 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      - run: pnpm exec wrangler deploy
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

Remote migrations are never run by CI; they're a deliberate, backed-up step ([§5](#5-remote-migrations-with-a-backup)).

## 2. Environments

| Environment | Where | Data | Switches |
| --- | --- | --- | --- |
| **Local** | `pnpm dev` (http://localhost:5173, app + API on one port) | `.wrangler/state` (local D1/R2), seeded with `scripts/seed-local.sql` | `.dev.vars`: `ENVIRONMENT=development`, demo logins allowed |
| **Test** | `pnpm test` (workerd, in-memory) | Fresh per test file | `ENVIRONMENT=test` set in `vitest.config.ts` |
| **Demo** (optional) | A separate Worker, `env.demo` in `wrangler.jsonc` (`{{PROJECT_SLUG}}-demo`) | Its own D1 and R2, demo data only | `DEMO_MODE=true`, signup off |
| **Production** | `{{LIVE_URL}}` | The real D1 and R2 | `ENVIRONMENT=production`, `DEMO_MODE=false`, `SIGNUP_ENABLED=false` (invite-only) |

- Never point local or demo at the production database. Never seed production with test data.
- Deploy the demo with `pnpm build && npx wrangler deploy --env demo`; it has its own secrets (`wrangler secret put NAME --env demo`).

## 3. First deploy checklist

With the owner, once:

1. **Account:** `npx wrangler login` with the project's Cloudflare account; `npx wrangler whoami` shows the right account.
2. **Resources** (only those the app uses):
   - `npx wrangler d1 create {{PROJECT_SLUG}}` → put the `database_id` in `wrangler.jsonc`.
   - `npx wrangler r2 bucket create {{PROJECT_SLUG}}-files`.
   - KV, Queues, Vectorize as needed ([`BACKEND_CLOUDFLARE.md` §10](./BACKEND_CLOUDFLARE.md#10-wranglerjsonc-reference)).
   - `pnpm cf-typegen`, commit the updated types.
3. **Migrations:** `pnpm db:migrate:remote` on the empty database (no backup needed the first time; confirm it's empty with `npx wrangler d1 execute DB --remote --command "SELECT name FROM sqlite_master"`).
4. **Secrets:** `npx wrangler secret put NAME` for each secret listed in `.dev.vars.example` (e.g. `ANTHROPIC_API_KEY`). Set a spend limit in the provider's console.
5. **Vars:** review `vars` in `wrangler.jsonc`: `ENVIRONMENT=production`, every demo or test switch off, allowed domains and extension ids correct.
6. **First admin:** create it with the project's one-off script (prints a temporary password once), not by opening signup.
7. **Deploy:** `pnpm run deploy` (not `pnpm deploy`).
8. **Smoke test** ([§8](#8-smoke-tests)), then record the live URL in `CLAUDE.md` and `docs/GUIDE.md`, and log the release in `docs/ROADMAP.md`.
9. **Observability:** confirm logs arrive (`npx wrangler tail`).

## 4. Every release

1. The milestone PR is merged into `main` after CI, `code-reviewer`, `ux-reviewer` (UI) and `qa-tester`.
2. Pending migrations? Back up and apply them first ([§5](#5-remote-migrations-with-a-backup)), with the owner. Migrations must be backward compatible with the code that's live (add columns and tables first; remove old ones in a later release), so the order "migrate, then deploy" never breaks the running version.
3. `pnpm run deploy` from a clean `main` checkout (`git status` empty).
4. Smoke test; watch `wrangler tail` for a few minutes.
5. Log the release (date, commit, migrations) in `docs/ROADMAP.md` and move features to "Released" (a feature is released only once merged, migrated and deployed).

## 5. Remote migrations with a backup

Only with the owner's go-ahead.

```bash
# 1. Which migrations are pending?
npx wrangler d1 migrations list DB --remote

# 2. Back up (schema + data) to a gitignored folder.
mkdir -p backups
npx wrangler d1 export {{PROJECT_SLUG}} --remote --output "backups/$(date +%Y%m%d-%H%M)-before-migrate.sql"

# 3. Note the time for Time Travel (D1 can restore to any point in the last 30 days).
date -u +%Y-%m-%dT%H:%M:%SZ

# 4. Apply.
pnpm db:migrate:remote

# 5. Check.
npx wrangler d1 migrations list DB --remote
npx wrangler d1 execute DB --remote --command "SELECT COUNT(*) FROM users"
```

If a migration fails or the data looks wrong: stop, don't deploy, and restore with `npx wrangler d1 time-travel restore {{PROJECT_SLUG}} --timestamp=<time from step 3>` (or import the export into a new database). Migrations that rewrite or drop data (a table becoming a view, a column moving) are tested on a production-like copy first ([`TESTING.md` §3](./TESTING.md#3-migration-tests)).

## 6. Secrets

Every name and its value per environment: [`ENV_AND_SECRETS.md`](./ENV_AND_SECRETS.md).

- Production: `npx wrangler secret put NAME` (prompts for the value; never put it on the command line or in chat). List with `npx wrangler secret list`.
- Local: `.dev.vars` (gitignored), documented by `.dev.vars.example` (names only).
- CI: GitHub repository or environment secrets, scoped tokens only.
- Rotate: put the new value, deploy if code changed, revoke the old one at the provider. Rotate immediately if a value was printed, pasted or committed.
- Never in `vars`, `VITE_*`, logs, git, screenshots or reports.

## 7. Rollback

- **Code:** `npx wrangler deployments list` shows recent versions; `npx wrangler rollback [<version-id>]` returns to a previous one in seconds. Then fix forward on a branch.
- **Database:** code rollback doesn't undo migrations. That's why migrations stay backward compatible ([§4](#4-every-release)). To undo data changes, use D1 Time Travel to the timestamp taken before migrating.
- **Config:** `vars` changes ship with a deploy, so a rollback restores them; secrets don't roll back (set the old value again if needed).
- Record every rollback and its cause in `docs/ROADMAP.md` (log and risks).

## 8. Smoke tests

Right after every deploy, on the live URL (by hand or with a small Playwright script, text checks only):

- [ ] `curl -s {{LIVE_URL}}/api/health` answers `{"ok":true,…}` with `Cache-Control: private, no-store`.
- [ ] `/` loads with no console errors; a deep link (e.g. `/items/x`) loads the app (SPA fallback), and an unknown path shows "Page not found".
- [ ] Sign in, see the main screen, sign out (signed-out API calls answer 401 and the app goes to `/login`).
- [ ] The main flow works end to end once (create, view, edit), then is cleaned up.
- [ ] Light and dark mode; one phone width.
- [ ] AI feature answers (if any), and its quota message appears when exhausted.
- [ ] No demo switches visible in production.

## 9. Monitoring and logs

- `"observability": { "enabled": true }` in `wrangler.jsonc` keeps Workers Logs (searchable in the dashboard: Workers & Pages → the Worker → Observability).
- Live: `npx wrangler tail` (add `--status error` or `--search "Unhandled"` to filter).
- Log one line per notable event (`Unhandled API error: <message>`, `Claude request failed 529`, `Deleted 12 unattached file(s)`), never personal data, prompts, tokens or secrets.
- Watch after each release: error rate, 5xx count, p95 latency, cron results, AI spend in the provider console.
- Performance after deploy: the `cloudflare:web-perf` skill (Core Web Vitals).

## 10. Known problems

| Problem | Fix |
| --- | --- |
| `pnpm install` fails with "Ignored build scripts: workerd" | `allowBuilds` in `pnpm-workspace.yaml` must have `workerd: true` and `esbuild: true` |
| `pnpm deploy` doesn't deploy | Use `pnpm run deploy` (`pnpm deploy` is a different pnpm command) |
| Wrangler asks to log in | `npx wrangler login` with the project's account |
| Types out of date after a config change | `pnpm cf-typegen`, commit `worker/worker-configuration.d.ts` |
| `gh: command not found` | Install the GitHub CLI and `gh auth login`, or give the owner the compare URL to open the PR |
| Dev server stops answering on `127.0.0.1` | Use http://[::1]:5173 or restart `pnpm dev` |
| A deploy works locally but 500s live | `npx wrangler tail`; usually a missing secret or a `vars` difference |
