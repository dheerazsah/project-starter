# {{PROJECT_NAME}}

{{ONE_PARAGRAPH_SUMMARY}}: what the app is, who uses it and the one job it does for them (see `docs/GUIDE.md`). Roles: {{ROLES}}. Sections: Overview (`/`, `src/features/overview/`), States (`/states`, `src/features/states/`; an example page, remove it once real pages exist) and {{SECTIONS}}. Routing is a tiny pathname router in `src/lib/route.ts` (`parseRoute`, chosen in `src/App.tsx`); sections are listed in `NAV` in `src/lib/nav.ts`. API calls go through `src/lib/api.ts` with types from `shared/api.ts`; GET data is shared through `useResource`/`invalidate` in `src/lib/resource.ts`. Brand colour: {{BRAND_COLOR}} (`--primary` in `src/index.css`); buttons have 10 px corners, never pills.

## Stack
- Vite + React 19 + TypeScript, Tailwind CSS v4, shadcn/ui (`base-nova`, built on Base UI, not Radix; all components in `src/components/ui/`), pnpm.
- API: a Cloudflare Worker with Hono (`worker/index.ts`) serving `/api/*`; everything else is the React app as static assets (`wrangler.jsonc`). Optional D1 (database) and R2 (files), commented out in `wrangler.jsonc` until needed.
- Live: {{LIVE_URL}}
- Theme tokens and fonts in `src/index.css`. Pages are chosen by path in `src/App.tsx`.

## Commands
| Task | Command |
| --- | --- |
| Dev server, app and API (http://localhost:5173) | `pnpm dev` |
| Typecheck · lint · test · build | `pnpm typecheck` · `pnpm lint` · `pnpm test` · `pnpm build` |
| Format | `pnpm format` |
| Worker types after editing `wrangler.jsonc` | `pnpm cf-typegen` |
| Local database migrations (once D1 is on) | `pnpm db:migrate:local` |
| Deploy (ask the owner first) | `pnpm run deploy` |

## Team (subagents in `.claude/agents/`)
`project-manager` plans and tracks · `product-designer` shapes and designs (WCAG 2.2 AA) · `developer` builds the front end · `backend-developer` builds the Worker API, data and sign-in · `ux-reviewer` audits screens against the UI principles and UX laws (read-only) · `code-reviewer` reviews diffs · `qa-tester` tests in the browser · `devops` builds, CI and deploys · `credits-optimizer` plans each task for full quality with no wasted credits and audits waste afterwards (read-only). Use `code-reviewer` before every commit, and `ux-reviewer` plus `qa-tester` after every UI change. The full process is in `docs/WORKFLOW.md`.

## Docs
| File | What it's for |
| --- | --- |
| `docs/GUIDE.md` | Reference: product, roles, features (F-IDs), architecture, develop, deploy. Update only when behaviour, permissions, routes or setup change. |
| `docs/ROADMAP.md` | The plan and the **only** place for status: dashboard, queue, decisions, risks, log. Update after every task. |
| `docs/DESIGN_SYSTEM.md` | This app's design rules: UI principles (§1), UX laws (§3), components, motion, accessibility, copy, the owner's standing feedback (§22) and the **UX/UI review checklist (§23)**. Read it before any UI work. |
| `docs/WORKFLOW.md` | The agentic process: who does what, in which order, what the owner approves, commits, seed and demo data, handoffs after a reset. |
| `docs/RESEARCH.md` | Competitor and UX research template; run it before big UI work. |
| `docs/BACKEND_CLOUDFLARE.md` | Worker, D1, R2, KV, Durable Objects, Workers AI vs Claude, cron, config and secrets, with code. |
| `docs/ENV_AND_SECRETS.md` | Every var and secret per environment (local, test, demo, production) and how to set it. Add a row with every new setting. |
| `docs/SECURITY.md` | Auth, sessions, CSRF, rate limits, authorization, tokens, uploads, AI prompt injection, headers, pre-PR checklist. |
| `docs/AI_FEATURES.md` | Grounded answers with citations, structured output, quotas and refunds, model choice, cost controls. |
| `docs/TESTING.md` | The Worker test harness, what to test, browser QA and accessibility checks. |
| `docs/DEPLOY_AND_CI.md` | CI, environments, first deploy, remote migrations with backups, rollback, monitoring. |
| `docs/BROWSER_EXTENSION.md` | Optional: an MV3 extension that talks to the Worker. |

After a context reset, read in order: `docs/GUIDE.md` → `docs/ROADMAP.md` (dashboard and the active task) → this file → `docs/DESIGN_SYSTEM.md` before UI work → `git status --short`.

## Rules
- Branches: `dksk/<type>/<name>`, lowercase kebab-case (`feat fix bugfix refactor chore docs test update`). Open PRs; don't push to `main` directly.
- Don't commit, push or deploy unless the owner asks. The owner also approves remote migrations, secrets, production data changes and local data resets.
- One build agent at a time, in the roadmap's order. Each build task ends with `code-reviewer`, then (UI) `ux-reviewer` and `qa-tester`.
- One commit per task, and every commit typechecks on its own.
- Status lives only in `docs/ROADMAP.md`. Big UI work starts with research (`docs/RESEARCH.md`).
- Seed and demo data come from real public content, never invented; one demo account per role behind `DEMO_MODE`, which is never on in production.
- Check every UI change at phone, tablet and desktop widths, in light and dark mode, with no console errors, and against the §23 checklist in `docs/DESIGN_SYSTEM.md`.
- Never reset, check out, stash or clean uncommitted work without asking.

## Working rules
- **Quality first.** Cut wasted credits (rework, re-reading, repeated full runs, oversized briefs, parallel edits that collide), never quality: design first where UI needs it, review and QA after every build, accessibility, tests.
- **Never PNG.** Verify with text checks first (DOM text, roles, computed styles, overflow, axe, console errors). Screenshots only when text can't tell, as JPEG at quality 60 with CSS scale (`type: "jpeg", quality: 60, scale: "css"`), and only a few per task.
- **Keep every shadcn component** in `src/components/ui/` and every brand file in `public/brand/`, even if unused. Re-add a component with `pnpm dlx shadcn@latest add <name> --overwrite` rather than hand-editing it, unless the fix belongs there.
- Ask the owner before adding a dependency.
- Strict types: no `any`, no `@ts-ignore` without a comment saying why.
- Theme tokens only: no hard-coded colours or fonts, no new vanilla `.css` files.
