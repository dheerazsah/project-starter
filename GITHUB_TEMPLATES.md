# GitHub templates

Copy-ready files for the new repo's `.github/` folder. During setup, Claude Code writes each block below to the path in its heading (SETUP.md step 11), then this file can be deleted from the repo. Replace `{{GITHUB_USERNAME}}` with the owner's GitHub handle.

| Path | What it does |
| --- | --- |
| `.github/pull_request_template.md` | Pre-fills every PR description |
| `.github/ISSUE_TEMPLATE/bug.yml` | Bug report form |
| `.github/ISSUE_TEMPLATE/task.yml` | Feature or task form (matches a roadmap row) |
| `.github/ISSUE_TEMPLATE/config.yml` | Turns off blank issues |
| `.github/CODEOWNERS` | Requests the owner's review on every PR |

Status lives in `docs/ROADMAP.md`, not in issues. Use issues for bug reports and ideas from outside the team; `project-manager` copies accepted ones into the roadmap queue and links the issue.

#### `.github/pull_request_template.md`

The outer fence has four backticks because the template contains a code block; copy only what's inside it.

````markdown
## Summary

<!-- One or two sentences: what this PR does and why. Milestone and task IDs from docs/ROADMAP.md. -->

Milestone: M?  ·  Tasks: M?-?, M?-?

## Screenshots

<!-- A few JPEGs from qa-tester (quality 60, CSS scale). Never PNG. Phone and desktop, light and dark where the change is visual. Delete this section for backend-only PRs. -->

## What changed

<!-- One line per commit: git log main..HEAD --oneline -->

- 

## How to verify

```bash
pnpm install --frozen-lockfile
pnpm typecheck
pnpm lint
pnpm test
pnpm build
```

Then: <!-- the manual steps: pages to open, roles to sign in as, what you should see -->

## Migrations, secrets and config

<!-- New migrations (file names), new vars or secrets (names only; listed in docs/ENV_AND_SECRETS.md), wrangler.jsonc changes. Write "None" if none. -->

## Checklist

- [ ] Branch is `dksk/<type>/<name>`; one commit per task, each typechecks on its own
- [ ] `code-reviewer` passed; findings fixed
- [ ] UI: `ux-reviewer` and `qa-tester` passed at phone, tablet and desktop, light and dark, no console errors, zero axe violations
- [ ] Security checklist in `docs/SECURITY.md` §10 (if `worker/`, auth, files or AI changed)
- [ ] New settings in `.dev.vars.example` and `docs/ENV_AND_SECRETS.md`
- [ ] `docs/ROADMAP.md` updated; `docs/GUIDE.md` updated if behaviour changed
- [ ] No secrets, personal data or PNGs in the diff
````

#### `.github/ISSUE_TEMPLATE/bug.yml`

```yaml
name: Bug
description: Something doesn't work as it should
labels: [bug]
body:
  - type: textarea
    id: what
    attributes:
      label: What happened
      description: What you did, what you expected, and what happened instead.
    validations:
      required: true
  - type: textarea
    id: steps
    attributes:
      label: Steps to reproduce
      placeholder: |
        1. Sign in as …
        2. Open …
        3. Press …
    validations:
      required: true
  - type: dropdown
    id: where
    attributes:
      label: Where
      options: [Production, Demo, Local]
    validations:
      required: true
  - type: input
    id: device
    attributes:
      label: Device, browser, width and theme
      placeholder: e.g. iPhone Safari, 375 px, dark
  - type: dropdown
    id: severity
    attributes:
      label: Severity
      options:
        - Blocker (can't do the job, data at risk)
        - Major (a workaround exists)
        - Minor (cosmetic or rare)
  - type: textarea
    id: extra
    attributes:
      label: Console errors or anything else
      description: Paste text, not screenshots, where you can. Remove personal data.
```

#### `.github/ISSUE_TEMPLATE/task.yml`

```yaml
name: Feature or task
description: A change to plan into the roadmap
labels: [task]
body:
  - type: textarea
    id: problem
    attributes:
      label: Problem
      description: Who is it for, what are they trying to do, and what's hard today?
    validations:
      required: true
  - type: textarea
    id: done
    attributes:
      label: Done when
      description: Checkable acceptance criteria, as they'll appear in docs/ROADMAP.md.
      placeholder: |
        - …
        - Checked at phone, tablet and desktop, light and dark
    validations:
      required: true
  - type: dropdown
    id: area
    attributes:
      label: Area
      options: [Front end, Back end, Both, Design, Docs, CI and deploy]
  - type: dropdown
    id: size
    attributes:
      label: Size (a guess is fine)
      options: [S (under half a day), M (about a day), L (several days)]
  - type: textarea
    id: notes
    attributes:
      label: Notes, links, research
```

#### `.github/ISSUE_TEMPLATE/config.yml`

```yaml
blank_issues_enabled: false
```

#### `.github/CODEOWNERS`

```text
# The owner reviews every PR. Add more lines (most specific last) as the team grows, e.g.
# /worker/      @{{GITHUB_USERNAME}}
# /migrations/  @{{GITHUB_USERNAME}}
*  @{{GITHUB_USERNAME}}
```

CODEOWNERS only requests reviews; to require them, the owner turns on branch protection for `main` ("Require review from Code Owners", "Require status checks: CI"). That's an owner action (`WORKFLOW.md` §3).
