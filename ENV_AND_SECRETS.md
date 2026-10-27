# Env vars and secrets

Every setting the app reads, where each one lives in every environment, and how to set it. This is the inventory; the patterns behind it are in [`BACKEND_CLOUDFLARE.md` §8](./BACKEND_CLOUDFLARE.md#8-env-vars-vs-secrets), the deploy steps in [`DEPLOY_AND_CI.md`](./DEPLOY_AND_CI.md). **Add a row here in the same change that adds a setting to the code.**

## 1. Three kinds of setting

| Kind | Lives in | Set with | Rolls back with a deploy? |
| --- | --- | --- | --- |
| **Var** (not secret) | `wrangler.jsonc` `vars` (top level = production, `env.demo.vars` = demo); locally `.dev.vars` | Edit the file, deploy | Yes |
| **Secret** | Cloudflare (encrypted per Worker and environment); locally `.dev.vars` (gitignored) | `npx wrangler secret put NAME` (add `--env demo` for demo) | No: set the old value again |
| **Client config** | Source code or `VITE_*` | Edit, build | Yes. **Always public**: everything in the Vite bundle can be read by anyone |

Rules:
- `.dev.vars.example` lists **every** name the Worker reads (values empty for secrets). `.dev.vars` is the local copy and is never committed.
- `wrangler secret put` prompts for the value. Never put a value on the command line, in chat, in a report, a screenshot or a log.
- Typed: vars come from `pnpm cf-typegen`; optional secrets are added by hand to `worker/env.d.ts`.
- Booleans are read tolerantly and default to the **safe** value when missing (`envFlag` in `BACKEND_CLOUDFLARE.md` §8).
- The tests set every binding they need in `vitest.config.ts`, so they never depend on `.dev.vars` ([`TESTING.md` §1](./TESTING.md#1-the-worker-test-harness)).

## 2. Inventory

Fill in the rows the app uses; delete the rest. "Owner" means the owner approves adding or changing it.

| Name | Kind | What it does | Local (`.dev.vars`) | Test (`vitest.config.ts`) | Demo (`env.demo`) | Production |
| --- | --- | --- | --- | --- | --- | --- |
| `ENVIRONMENT` | Var | `development`, `test`, `demo` or `production`; test-only hooks work only in `test` | `development` | `test` | `demo` | `production` |
| `DEMO_MODE` | Var | One-click demo sign-ins and the demo banner | `true` if wanted | `false` | `true` | **`false`, always** |
| `SIGNUP_ENABLED` | Var | Open self sign-up | `true` | `true` | `false` | `false` (invite-only) unless the owner decides otherwise |
| `ALLOWED_EMAIL_DOMAINS` | Var | Comma-separated domains allowed to sign up | any | set per test | as production | `{{EMAIL_DOMAIN}}` |
| `ANTHROPIC_MODEL` | Var | Model id for AI features (check the `claude-api` skill for current ids) | as production | unused (stub client) | as production | the chosen id |
| `EXTENSION_IDS` | Var | Browser extension ids allowed by CORS ([`BROWSER_EXTENSION.md`](./BROWSER_EXTENSION.md)) | dev build id | set per test | demo build id | store id |
| `ANTHROPIC_API_KEY` | Secret | Claude API; AI features degrade without it | own dev key, low spend limit | `""` | own key | own key, spend limit set |
| `RESEND_API_KEY` (or the mail provider's) | Secret | Sending email | empty (log instead) | `""` | empty or own key | own key |
| Signing keys (e.g. `SESSION_SECRET`) | Secret | Signing tokens or cookies, if the auth design needs one | any random value | fixed test value | own value | own value, rotated on leak |
| `CLOUDFLARE_API_TOKEN` | GitHub secret | CI deploy only, when the owner turns it on (scoped to "Edit Cloudflare Workers" on one account; [`DEPLOY_AND_CI.md` §1](./DEPLOY_AND_CI.md#1-ci-with-github-actions)) | none | none | GitHub environment `demo` | GitHub environment `production` |
| `CLOUDFLARE_ACCOUNT_ID` | GitHub secret or var | CI deploy only | none | none | as production | the account id |

Bindings (`DB`, `FILES`, `KV`, Durable Objects) aren't settings, but they differ per environment too: local is `.wrangler/state`, tests are in-memory, demo and production each have **their own** D1 database and R2 bucket ([`BACKEND_CLOUDFLARE.md` §10](./BACKEND_CLOUDFLARE.md#10-wranglerjsonc-reference)). Bindings are not inherited by `env.demo`: repeat them there.

## 3. Demo vs production

| | Demo | Production |
| --- | --- | --- |
| Worker | `{{PROJECT_SLUG}}-demo` (`env.demo`) | `{{PROJECT_SLUG}}` |
| D1 / R2 | `{{PROJECT_SLUG}}-demo` / `{{PROJECT_SLUG}}-demo-files` | `{{PROJECT_SLUG}}` / `{{PROJECT_SLUG}}-files` |
| Data | Seeded from real public content, reset freely with the owner's OK | Real data only; never seeded with demo or test data |
| Sign-in | Pre-filled demo account per role (`DEMO_MODE=true`), signup off | Real accounts; `DEMO_MODE=false`, signup per the owner |
| Secrets | Its own (`--env demo`), lower spend limits | Its own; never shared with demo |
| Deploy | `pnpm build && npx wrangler deploy --env demo` | `pnpm run deploy` |

Never point local or demo at the production database. A smoke test after every production deploy confirms no demo switch is visible ([`DEPLOY_AND_CI.md` §8](./DEPLOY_AND_CI.md#8-smoke-tests)).

## 4. Common tasks

| Task | Command or step | Owner approval |
| --- | --- | --- |
| Start local work | `cp .dev.vars.example .dev.vars`, fill in local values | No |
| Add a var | Add to `wrangler.jsonc` `vars` (and `env.demo.vars`), `.dev.vars.example`, this table; `pnpm cf-typegen` | Ships with the next deploy |
| Add a secret | Name in `.dev.vars.example` and `worker/env.d.ts`; then `npx wrangler secret put NAME` (and `--env demo`) | Yes |
| List secrets (names only) | `npx wrangler secret list` (`--env demo`) | No |
| Rotate a secret | `npx wrangler secret put NAME` with the new value, then revoke the old one at the provider | Yes |
| Remove a secret | `npx wrangler secret delete NAME` after the code no longer reads it | Yes |
| CI deploy secrets | GitHub → Settings → Environments → `production` / `demo` → secrets | Yes |

A deploy that works locally but returns 500 live is usually a missing secret or a `vars` difference: compare `npx wrangler secret list` with this table.
