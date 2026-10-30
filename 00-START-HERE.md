# Start here: the project starter

A flat set of Markdown files that turns an empty folder into a production-ready project on Vite + React 19 + TypeScript + Tailwind CSS v4 + shadcn/ui (Base UI), with a Hono API on Cloudflare Workers, a full design system, the docs set, and a team of Claude Code agents that already know the conventions.

**Self-contained, works from any location.** These files don't link to anything outside themselves and contain no machine-specific paths. Copy them anywhere, paste them into any IDE, and they still work. Links between them are relative to where they end up in the new repo (below).

## How to use them in a new project

1. Create an empty folder for the project and open it in the IDE.
2. Paste all of these `.md` files into it (top level is fine), **except** `README.md`, `LICENSE.md` and `CHANGELOG.md`, which describe the starter's own repo.
3. Ask Claude Code: **"Set up the project from SETUP.md"**. It will:
   - move each file to its place (table below),
   - scaffold and configure the app by following `SETUP.md` step by step (including the `.github/` files from `GITHUB_TEMPLATES.md`),
   - run the verify step (install, typecheck, lint, React Doctor, test, build) and report.
4. Fill in the placeholders (table below) with the owner; that's roadmap task M0-1.
5. Delete `00-START-HERE.md`, `SETUP.md` and `GITHUB_TEMPLATES.md` from the repo afterwards, or keep a copy outside it for the next project.

### Step 0: where each file goes

| File | Goes to | What it is |
| --- | --- | --- |
| `00-START-HERE.md` | stays outside the repo (delete after setup) | This guide |
| `SETUP.md` | stays outside the repo (delete after setup) | **A recipe for Claude Code**: scaffold commands, every config file, the theme tokens, the app shell, lib helpers, the Worker with `/api/health` and its test, CI, verify |
| `GITHUB_TEMPLATES.md` | stays outside the repo (delete after setup); its blocks go to `.github/` | PR template, issue forms, CODEOWNERS, each with its path |
| `README.md`, `LICENSE.md`, `CHANGELOG.md` | not pasted | The starter's own repo page, license and version history |
| `CLAUDE.md` | repo root: `CLAUDE.md` | What every agent reads first: stack, commands, team, docs, rules |
| `GUIDE.md` | `docs/GUIDE.md` | Reference template: product, problem, sitemap, journeys, roles, features (F-IDs), architecture, develop, deploy |
| `ROADMAP.md` | `docs/ROADMAP.md` | The plan and the only status source: dashboard, queue, decisions, risks, log (M0 is pre-filled) |
| `DESIGN_SYSTEM.md` | `docs/DESIGN_SYSTEM.md` | UI principles (§1), UX laws (§3), tokens, components, motion, accessibility, copy, owner feedback, and the **UX/UI review checklist (§23)** |
| `WORKFLOW.md` | `docs/WORKFLOW.md` | The agentic process: who does what, in which order, what the owner approves, commits, demo data, handoffs, lessons |
| `RESEARCH.md` | `docs/RESEARCH.md` | Competitor and UX research template for big UI work: questions, output format, what not to copy |
| `BACKEND_CLOUDFLARE.md` | `docs/BACKEND_CLOUDFLARE.md` | Worker + Hono, D1, R2, KV vs DO, Durable Objects and WebSockets, Workers AI vs Claude, cron, vars vs secrets, a full `wrangler.jsonc` reference; migrations, seeds and demo accounts (§2) |
| `ENV_AND_SECRETS.md` | `docs/ENV_AND_SECRETS.md` | Every var and secret, local vs test vs demo vs production, how to set each, demo vs production split |
| `SECURITY.md` | `docs/SECURITY.md` | Own accounts, sessions, CSRF, rate limits, authz/IDOR, tokens and CORS, uploads, AI prompt injection, headers, pre-PR checklist |
| `AI_FEATURES.md` | `docs/AI_FEATURES.md` | Grounded answers with citations, structured output, quotas and refunds, small talk, model choice, cost controls |
| `TESTING.md` | `docs/TESTING.md` | The Worker test harness (in-memory D1), what to test, migration tests, Playwright text-first QA, axe |
| `DEPLOY_AND_CI.md` | `docs/DEPLOY_AND_CI.md` | GitHub Actions CI, environments, first deploy, remote migrations with backups, secrets, rollback, smoke tests, monitoring |
| `BROWSER_EXTENSION.md` | `docs/BROWSER_EXTENSION.md` (only if the product has an extension) | Optional module: an MV3 extension that talks to the Worker |
| `agent-project-manager.md` | `.claude/agents/project-manager.md` | Plans, keeps GUIDE and ROADMAP current, branches and PR text |
| `agent-credits-optimizer.md` | `.claude/agents/credits-optimizer.md` | Plans each task for full quality without waste; audits waste afterwards |
| `agent-product-designer.md` | `.claude/agents/product-designer.md` | Brief, flow, scope, screens; WCAG 2.2 AA; owns `DESIGN_SYSTEM.md` |
| `agent-developer.md` | `.claude/agents/developer.md` | Builds the front end in `src/` |
| `agent-backend-developer.md` | `.claude/agents/backend-developer.md` | Builds the Worker API, data, files, sign-in, AI calls |
| `agent-ux-reviewer.md` | `.claude/agents/ux-reviewer.md` | Audits screens against the UI principles, UX laws and §23 checklist (read-only) |
| `agent-code-reviewer.md` | `.claude/agents/code-reviewer.md` | Reviews diffs before every commit (read-only) |
| `agent-qa-tester.md` | `.claude/agents/qa-tester.md` | Tests in the browser: widths, themes, keyboard, axe; text checks first, JPEG only |
| `agent-devops.md` | `.claude/agents/devops.md` | Builds, CI, secrets, deploys (asks before production) |

The agent files keep their frontmatter (`name`, `description`, `tools`), so they work as soon as they're in `.claude/agents/`; the file name without `agent-` must match `name`.

Shell version of step 0 (from the folder with the pasted files):

```bash
mkdir -p docs .claude/agents
mv GUIDE.md ROADMAP.md DESIGN_SYSTEM.md WORKFLOW.md RESEARCH.md BACKEND_CLOUDFLARE.md ENV_AND_SECRETS.md SECURITY.md AI_FEATURES.md TESTING.md DEPLOY_AND_CI.md docs/
[ -f BROWSER_EXTENSION.md ] && mv BROWSER_EXTENSION.md docs/
for f in agent-*.md; do mv "$f" ".claude/agents/${f#agent-}"; done
```

`GITHUB_TEMPLATES.md` stays at the root until SETUP.md step 11 has written its blocks to `.github/`.

## Placeholders

Replace these after setup (find what's left with `grep -rnE "\{\{[A-Z0-9_]+\}\}" --exclude-dir=node_modules .`):

| Placeholder | Where | What to put |
| --- | --- | --- |
| `{{PROJECT_SLUG}}` | `package.json` `name`, `wrangler.jsonc` `name` and resources, `src/lib/app.ts` (`STORAGE_PREFIX`), docs | The project slug, lowercase kebab-case, e.g. `team-planner` |
| `{{PROJECT_NAME}}` | `src/lib/app.ts`, `index.html`, `CLAUDE.md`, `docs/*`, `.claude/agents/*` | The product name shown to people |
| `{{PROJECT_DESCRIPTION}}` | `index.html` | One line for the meta description |
| `{{ONE_PARAGRAPH_SUMMARY}}`, `{{ONE_LINE_SUMMARY}}`, `{{ROLES}}`, `{{SECTIONS}}` | `CLAUDE.md`, agents | What the app does, its roles and sections |
| `{{BRAND_COLOR}}` | `src/index.css` (every value marked `BRAND`), `index.html` (`theme-color`), `public/favicon.svg`, `docs/DESIGN_SYSTEM.md` | The brand colour; in CSS as oklch |
| `{{BRAND_PALETTE}}`, `{{FONT}}`, `{{CONTRAST_TABLE}}`, `{{GLOSSARY}}` | `docs/DESIGN_SYSTEM.md`, `src/index.css` (`--font-sans`) | Brand colours, UI font, measured contrast, one word per concept |
| `{{LOGO}}` | `public/favicon.svg`, `public/brand/`, `src/components/app-shell.tsx` (`HomeLink`) | Logo and mark (with a dark-mode version) |
| `{{LIVE_URL}}`, `{{GITHUB_REPO}}`, `{{OWNER_NAME}}` | `CLAUDE.md`, `docs/GUIDE.md`, `docs/DEPLOY_AND_CI.md`, agents | Live URL, `org/repo`, the maintainer |
| `{{AUTH_APPROACH}}` | `.claude/agents/backend-developer.md` | How people sign in (see `docs/SECURITY.md` §1) |
| `{{EMAIL_DOMAIN}}` | `docs/ENV_AND_SECRETS.md` | The email domain allowed to sign up, if signup is limited |
| `{{GITHUB_USERNAME}}` | `.github/CODEOWNERS` (from `GITHUB_TEMPLATES.md`) | The owner's GitHub handle, without `@` |
| `{{ORG}}`, `{{TOPICS}}` | `docs/AI_FEATURES.md` prompts | The organisation and what the assistant answers about |
| `{{D1_DATABASE_ID}}` | `wrangler.jsonc` | Only once D1 is turned on |
| `{{DATE}}`, `{{GOAL_1}}`, `{{FEATURE}}`, `{{TACTIC_1}}`, `{{JOURNEY_NAME}}` and the rest | `docs/GUIDE.md`, `docs/ROADMAP.md`, `docs/RESEARCH.md` | The product, goals, features, journeys and plan (the templates explain each) |

The branch prefix `dksk/` is the maintainer's initials. Another maintainer can swap it in every file with one search and replace (`dksk/` → `<initials>/`) before setup.

## Conventions these files carry

- Branches `dksk/<type>/<name>`; one commit per task, each one typechecks. No commit, push, deploy, remote migration, secret change or local data reset without the owner.
- One build agent at a time; every build ends with `code-reviewer`, and UI builds with `ux-reviewer` and `qa-tester`.
- Status lives only in `ROADMAP.md` (status column right after #); `GUIDE.md` is reference.
- The UI principles and UX laws in `DESIGN_SYSTEM.md` are followed on every screen, checked with the §23 checklist. Big UI work starts with research (`RESEARCH.md`).
- Seed and demo data come from real public content, never invented; demo deployments have a pre-filled account per role; `DEMO_MODE` is off in production.
- Check every UI change at phone, tablet and desktop, light and dark, with no console errors.
- Quality first; cut wasted credits, not quality. Never PNG: text checks first, then at most a few JPEGs (quality 60, CSS scale).
- Keep every shadcn component and brand file, even unused ones. Buttons have 10 px corners and `font-medium`, never pills; theme tokens only.
