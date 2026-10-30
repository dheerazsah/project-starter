# {{PROJECT_NAME}} guide

How {{PROJECT_NAME}} works: the product, who can do what, every feature, the architecture, and how to develop, deploy and hand over work. **For what's being worked on now, see [`ROADMAP.md`](./ROADMAP.md).**

> **Template.** Replace each `{{…}}` and the *italic guidance* with real content. Delete the guidance once a section is filled in.

## Start here

| You want to… | Read |
| --- | --- |
| Know what's in progress, what's next, what needs the owner | [`ROADMAP.md`](./ROADMAP.md): the dashboard at the top |
| Understand a feature, who can use it and where its code is | [Features](#features) below |
| Set up, run or deploy the app | [Develop](#develop) and [Deploy](#deploy) |
| Pick up work after a context or token reset | [Working on the project](#working-on-the-project) |
| Change any screen | [`DESIGN_SYSTEM.md`](./DESIGN_SYSTEM.md) first |
| Follow repo conventions (branches, commands, agents) | [`CLAUDE.md`](../CLAUDE.md) |

**Two files, two jobs.** This guide is **reference**: update it only when behaviour, permissions, routes or setup change. [`ROADMAP.md`](./ROADMAP.md) is the plan and the **only** place for task status, owner decisions, risks and the log: update it after every task. Don't copy status between them.

## Product

*Two or three sentences: what the app is, who uses it, and the one job it does for them. Then what it replaces, if anything.*

{{PRODUCT_SUMMARY}}

### The problem it solves

*One sentence on the gap the product closes, then the tactics that close it. Example of the shape: "Getting people from curious to signed up", with tactics like "Try it before signing up" or "Limits that explain themselves".*

| Tactic | How {{PROJECT_NAME}} does it |
| --- | --- |
| {{TACTIC_1}} | {{HOW_1}} |
| {{TACTIC_2}} | {{HOW_2}} |

Scope for now: *what's deliberately left out, so nobody builds it by accident.*

### Goals

*Two to four goals, each with a measurable signal. Goals describe outcomes for people, not features.*

| Goal | How we measure it |
| --- | --- |
| {{GOAL_1}} | {{MEASURE_1}} (e.g. "a new person finds what they need within 1 minute") |
| {{GOAL_2}} | {{MEASURE_2}} |

### Principles

*The few rules that settle arguments. Keep the last one.*

- {{PRINCIPLE_1}} (e.g. "The main screen is the one people use daily; no landing page.")
- {{PRINCIPLE_2}} (e.g. "Nothing is published without an approval.")
- The server enforces every permission; the UI only hides what a role can't use.
- The UI follows [`DESIGN_SYSTEM.md`](./DESIGN_SYSTEM.md): calm, accessible (WCAG 2.2 AA), responsive, light and dark.

### Roles

*One row per role, most limited first. If there's only one kind of user, say so.*

| Role | Can |
| --- | --- |
| {{ROLE_1}} (e.g. member) | {{WHAT_THEY_CAN_DO}} |
| {{ROLE_2}} (e.g. admin) | Everything a {{ROLE_1}} can, plus {{EXTRA}} |

The UI hides what a role can't use, but the Worker enforces every rule.

### Sitemap

*Every screen, grouped. Overlays (dialogs, ⌘K, menus) count too.*

Public ({{PUBLIC_SCREENS}}, e.g. Log in, Reset password) · App ({{APP_SCREENS}}) · Admin ({{ADMIN_SCREENS}}) · Overlays ({{OVERLAYS}}, e.g. Search ⌘K, Settings).

### Journeys

*One table per important flow (first use, the core loop, recovery such as a forgotten password). For every "at risk" step, name the design response. See `DESIGN_SYSTEM.md` §3.*

**{{JOURNEY_NAME}}**

| Step | Where | Feels | Design response |
| --- | --- | --- | --- |
| {{STEP}} | {{SCREEN}} | good / unsure / at risk | {{RESPONSE}} (e.g. "Hits a limit → a friendly prompt about what an account adds, not an error") |

## Features

Feature IDs are permanent: never renumber or reuse one. **State** is one of: *Committed* (in git history), *Built, unreleased* (works locally, waiting for review, merge and deploy), *In progress*, *Planned*. Live status and order are in [`ROADMAP.md`](./ROADMAP.md).

*Format for every feature (copy it):*

```md
### F0N Feature name
- **State:** Planned. **Routes:** `/path`, `/path/:id`. **Who:** which roles.
- What it does, in one or two lines, and the rules that matter (limits, permissions, edge cases).
- **Code:** [`file.tsx`](../src/features/area/file.tsx), [`worker/routes/area.ts`](../worker/routes/area.ts); migration `000N`.
```

### F01 App shell, accessibility and motion
- **State:** Committed (from the starter). **Routes:** all. **Who:** everyone.
- Sidebar (`NAV`), phone menu, theme (light, dark, system), page titles, skip link, loading/empty/error states, route focus, toasts, and subtle motion that turns off with reduced motion. Checked at phone, tablet and desktop, light and dark (WCAG 2.2 AA).
- **Code:** [`app-shell.tsx`](../src/components/app-shell.tsx), [`page.tsx`](../src/components/page.tsx), [`states.tsx`](../src/components/states.tsx), [`index.css`](../src/index.css), [`motion.ts`](../src/lib/motion.ts), [`route-focus.ts`](../src/lib/route-focus.ts), [`nav.ts`](../src/lib/nav.ts).

### F02 API health
- **State:** Committed (from the starter). **Route:** `GET /api/health`. **Who:** anyone.
- Answers `{ ok, environment, time }`; the Overview page shows it. Every `/api/*` response is `Cache-Control: private, no-store`; errors are JSON `{ error }`.
- **Code:** [`worker/index.ts`](../worker/index.ts), [`shared/api.ts`](../shared/api.ts), [`worker/test/health.test.ts`](../worker/test/health.test.ts).

### F03 {{FIRST_REAL_FEATURE}}
- **State:** Planned. **Routes:** {{ROUTES}}. **Who:** {{ROLES}}.
- {{WHAT_IT_DOES}}
- **Code:** {{FILES}}

### Planned features
Details, order and status are in [`ROADMAP.md`](./ROADMAP.md).

| ID | Feature | Milestone |
| --- | --- | --- |
| F04 | {{FEATURE}} | M1 |

## Architecture

```text
Browser ── Cloudflare Worker
             ├─ /api/*  → Hono routes (worker/): permissions, data, files
             └─ else    → the React app (dist/, single-page app)
```

| Part | Where |
| --- | --- |
| React app | `src/` (pages chosen in `src/App.tsx`, routes in `src/lib/route.ts`, nav in `src/lib/nav.ts`) |
| Shared API types | `shared/api.ts` |
| Worker API | `worker/` (entry `worker/index.ts`, errors in `worker/http.ts`, tests in `worker/test/`) |
| Database schema | `migrations/` (D1, optional) |
| Uploaded files | R2 bucket `{{PROJECT_SLUG}}-files` (optional) |
| Config | `wrangler.jsonc`, `.dev.vars` (local, from `.dev.vars.example`) |

Stack: Vite, React 19, TypeScript, Tailwind CSS v4, Base UI/shadcn; Hono on Cloudflare Workers; {{DATA_AND_SERVICES}} (e.g. D1 for records, R2 for files, Claude for AI features).

### Why these choices

*One line per architecture decision and its reason, so nobody undoes it by accident (e.g. "A Durable Object per workspace: one ordered authority per room, and it hosts the WebSockets"; "D1 as the index, not the message store"; "Server-stamped identity: clients can't spoof authors or times"). Patterns are in [`BACKEND_CLOUDFLARE.md`](./BACKEND_CLOUDFLARE.md).*

- {{DECISION}}: {{REASON}}

*Describe the main data flow in one paragraph each (e.g. "Sign-in: …", "Saving an item: …"). Include security-relevant details: session cookie settings, password hashing, rate limits, origin checks.*

## Develop

| Task | Command |
| --- | --- |
| Run app and API (http://localhost:5173) | `pnpm dev` |
| Typecheck · lint · test · build | `pnpm typecheck` · `pnpm lint` · `pnpm test` · `pnpm build` |
| Worker types after editing `wrangler.jsonc` | `pnpm cf-typegen` |
| Local database: migrate (once D1 is on) | `pnpm db:migrate:local` |

Copy `.dev.vars.example` to `.dev.vars` (`ENVIRONMENT=development`) and add local secrets there; every setting is listed in [`ENV_AND_SECRETS.md`](./ENV_AND_SECRETS.md). Never commit `.dev.vars`, keys or `.pem` files. Local demo data: `scripts/seed-local.sql` (idempotent, real public content, one demo account per role; see [`BACKEND_CLOUDFLARE.md` §2](./BACKEND_CLOUDFLARE.md#seeds-and-demo-accounts)).

**Turning on D1:** `npx wrangler d1 create {{PROJECT_SLUG}}`, uncomment `d1_databases` in `wrangler.jsonc` with the ID, `pnpm cf-typegen`, write `migrations/0001_init.sql`, `pnpm db:migrate:local`. For tests, read the migrations in `vitest.config.ts` (`readD1Migrations("./migrations")` from `@cloudflare/vitest-plugin`, passed as a binding) and apply them in the test with `applyD1Migrations` from `cloudflare:test`.

**Browser checks:** Playwright is a devDependency. Install a browser once with `pnpm exec playwright install chromium` (or use an installed Chrome with `channel: "chrome"`). Text checks first; screenshots only as JPEG quality 60, never PNG.

## Deploy

Only with the owner's go-ahead. First deploy:

1. `npx wrangler login` with the project's Cloudflare account.
2. If the app uses D1 or R2: create them, put the real IDs in `wrangler.jsonc`, `pnpm cf-typegen`.
3. Back up the remote database (if any), then apply migrations: `pnpm db:migrate:remote`.
4. Add secrets: `npx wrangler secret put NAME` for each one in `.dev.vars.example` ([`ENV_AND_SECRETS.md`](./ENV_AND_SECRETS.md)).
5. Check the production `vars` in `wrangler.jsonc` (no demo or test switches on).
6. `pnpm run deploy`, open {{LIVE_URL}}, and smoke-test the main flow.

Never seed production with test data.

## Working on the project

**After a reset, read in order:** this guide → [`ROADMAP.md`](./ROADMAP.md) (dashboard and the active task) → [`CLAUDE.md`](../CLAUDE.md) → [`DESIGN_SYSTEM.md`](./DESIGN_SYSTEM.md) before UI work → `git status --short`.

**Rules**
- One build task at a time, in the roadmap's order. Each ends with `code-reviewer`, then (UI) `ux-reviewer` and `qa-tester`.
- Never reset, check out, stash or clean uncommitted work. If a task conflicts with it, stop and ask.
- Ask the owner before committing, pushing, opening a PR, deploying, running remote migrations, resetting local data or changing secrets.
- UI changes: phone, tablet and desktop, light and dark, keyboard and screen reader, no console errors.
- Implemented isn't released: a feature is released only once merged, migrated and deployed.

**When a task finishes**
1. In [`ROADMAP.md`](./ROADMAP.md): mark it ✅, update the dashboard, and add a log line (plus any decision or risk).
2. In this guide: only if behaviour, permissions, routes or setup changed. Update that feature's section and its **State**.

**Adding a feature:** take the next unused F number and add a section in the format above: State, Routes, Who, what it does and its rules, Code (with migrations).
