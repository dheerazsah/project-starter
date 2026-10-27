# Changelog

All notable changes to the starter. Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions follow [Semantic Versioning](https://semver.org/) (major: a file moves, is renamed or changes its contract with SETUP.md; minor: new files or sections; patch: fixes and wording).

## [1.0.0] - 2026-10-27

First public release.

### Added
- `00-START-HERE.md`: how to use the starter, where each file goes, placeholders, conventions.
- `SETUP.md`: a tested recipe for Claude Code that scaffolds Vite + React 19 + TypeScript + Tailwind CSS v4 + shadcn/ui (Base UI) with a Hono API on Cloudflare Workers, `/api/health` and its test, CI, and a verify step.
- Docs templates: `CLAUDE.md`, `GUIDE.md`, `ROADMAP.md`, `DESIGN_SYSTEM.md` (UI principles, UX laws, the §23 review checklist), `WORKFLOW.md`.
- Reference docs: `BACKEND_CLOUDFLARE.md`, `SECURITY.md`, `AI_FEATURES.md`, `TESTING.md`, `DEPLOY_AND_CI.md`, optional `BROWSER_EXTENSION.md`.
- `ENV_AND_SECRETS.md`: every var and secret per environment, demo vs production, how to set each.
- `RESEARCH.md`: competitor and UX research template (questions, output format, what not to copy).
- `GITHUB_TEMPLATES.md`: PR template, issue forms, CODEOWNERS, written to `.github/` during setup.
- Nine Claude Code agents (`agent-*.md`): project-manager, credits-optimizer, product-designer, developer, backend-developer, ux-reviewer, code-reviewer, qa-tester, devops.
- `WORKFLOW.md` lessons: one build agent at a time, status only in the roadmap, commits that typecheck one by one, seed and demo data from real public content, demo accounts per role, research before big UI work, owner approval points.
- `BACKEND_CLOUDFLARE.md` §2: migration numbering rules and the idempotent seed pattern.
- Repo files for the starter itself: `README.md`, `LICENSE.md` (MIT), this changelog.

### Fixed
- `SETUP.md` `.gitignore` now ignores `backups/` (remote database exports hold personal data; `SECURITY.md` §8 already assumed it).
