# Project starter (the starter's own repo)

This repo is the starter kit, not an app. `template/` is copied as is into every new project; `README.md`, `LICENSE.md`, `CHANGELOG.md` and this file describe the starter itself and are never copied. The app's own instructions are in `template/CLAUDE.md`. Don't follow them here; edit them as content.

## Rules
- Every file in `template/` sits at the path it keeps in a new project: `CLAUDE.md` and the setup-only files (`00-START-HERE.md`, `SETUP.md`, `GITHUB_TEMPLATES.md`) at its root, docs in `template/docs/`, agents in `template/.claude/agents/<name>.md` (file name = frontmatter `name`).
- Links between template files are relative to the file's own folder and never leave `template/`. No absolute or machine-specific paths, and no one project's names.
- A new or moved file updates the tables in `README.md` and `template/00-START-HERE.md`, the docs list in `template/SETUP.md` ("Before you start") and, if agents read it, the docs table in `template/CLAUDE.md`.
- Record every change in `CHANGELOG.md` (major: a file moves, is renamed or changes its contract with `SETUP.md`; minor: new files or sections; patch: fixes and wording).
- If a code block in `template/SETUP.md` changes, run the whole recipe in an empty folder (`cp -R template/. <dir>/`) and its verify step before merging.
- Branches `dksk/<type>/<name>`; open PRs, don't push to `main`. Don't commit or push unless the owner asks.
