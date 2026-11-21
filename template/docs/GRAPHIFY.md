# Graphify: a code map the agents check first

Without a map, an agent answers "where is X?" or "what breaks if I change Y?" by opening file after file until it finds the answer, and every file it opens costs tokens. [Graphify](https://github.com/Graphify-Labs/graphify) turns the code into a knowledge graph: every file, component, function, import and call, and how they connect. The agent asks the graph and opens only the lines it points to.

This file says when the map saves credits and when it doesn't, what it installs, and the rules agents follow. Checked on 21 November 2026 against `graphifyy` 0.9.114, tested on the app `SETUP.md` builds. Graphify changes fast, so reopen the repo before installing.

## Contents
1. [What it costs and saves](#1-what-it-costs-and-saves)
2. [When to use it, and when not](#2-when-to-use-it-and-when-not)
3. [What it installs and runs](#3-what-it-installs-and-runs)
4. [Install (with the owner's OK)](#4-install-with-the-owners-ok)
5. [Keeping the map current](#5-keeping-the-map-current)
6. [Settings, limits and removal](#6-settings-limits-and-removal)

## 1. What it costs and saves

Measured on the freshly set-up app (about 19k tokens of our own code in `src/`, `worker/` and `shared/`; the map has 379 nodes):

| Way to find something | Tokens read |
| --- | --- |
| `graphify path "WelcomePage()" "get()"`: how two things connect | about 50 |
| `graphify explain "ApiHealth()"`: one thing and what it touches | about 130 |
| `graphify query "<question>"`: a question in words | 1,000 to 2,000 (capped at about 2,000; set lower with `--budget`) |
| Opening the 4 files that answer one question | about 3,300 |
| Reading all our code to get oriented | about 19,000 |
| **Loading the `/graphify` skill** | **about 13,000 (a 50 KB file)** |

What that means:

- **Building and updating the map is free.** `graphify update .` parses the code on your computer with tree-sitter: no LLM, no credits, a few seconds. TypeScript, TSX and JavaScript are supported.
- **`path` and `explain` are almost free; `query` costs about one file.** A query only saves credits when it replaces opening two or more files.
- **The `/graphify` skill is expensive.** Don't type `/graphify` or let an agent load the skill for everyday questions: call the CLI commands above from Bash, which never loads it. The skill is only for mapping docs, PDFs and images, which this project doesn't need (§4).
- **The saving grows with the code.** On the starter skeleton an agent finds most things in a file or two, so the map saves little. Once the app has dozens of features, one `explain` replaces several file reads. `graphify benchmark` measures the saving for this project.
- **It doesn't optimize the code itself.** It makes agents cheaper at finding their way around. Code quality still comes from `code-reviewer`, React Doctor and the tests.

## 2. When to use it, and when not

Applies when `graphify-out/graph.json` exists. The cheapest tool that answers the question wins.

**Use the map:**

| Situation | Command | Instead of |
| --- | --- | --- |
| You know a name and need its callers, imports or what it touches (before changing a shared component, helper, hook or route) | `graphify explain "<Name>"` | Grepping for the name and opening each match |
| How a screen reaches the API, or how a route reaches the database | `graphify path "<A>" "<B>"` | Reading the chain file by file |
| You don't know where something lives ("where are sessions checked?") | `graphify query "<question>" --budget 800` | Listing folders and opening likely files |
| Orienting after a context reset, or briefing a builder (`credits-optimizer`, `project-manager`) | `explain` or `query` on the task's main names | Reading whole folders, or pasting long file lists |
| A bug that crosses layers (front end → `src/lib/api.ts` → Worker → D1) | `graphify path` between the two ends | Opening every layer in turn |

Names take the form the map stores: functions and components with `()` (`"ApiHealth()"`), types and constants without (`"HealthResponse"`). Exit code 3 means the name matched several things: be more specific. Exit code 4 means no match: use `grep`.

**Skip the map:**

- **You already know the file and lines:** read them. A query costs more than the read.
- **Exact text** (an error message, a CSS class, a copy string, a token name, a TODO): `grep`, which costs a few dozen tokens.
- **Questions about docs, rules or process:** the docs table in `CLAUDE.md` already says which doc to open; the map holds code only.
- **Small, one-file changes** (copy, a style, a single fix the owner pointed at).
- **The map is stale** (you pulled, switched branches or changed code and the git hooks aren't installed): run `graphify update .` first (free), or skip it this time.

**Always:**

- Open only the files and lines the answer names. Don't read `GRAPH_REPORT.md`, `graph.json` or `graph.html` whole; they're large.
- The map isn't the code. Read the real lines before you edit them, and check `INFERRED` edges (Graphify's guesses) against the source. `EXTRACTED` edges come straight from the code.
- After changing code, run `graphify update .` unless the git hooks are installed.
- `credits-optimizer` counts "opened files a 50-token `explain` would have pointed past" as waste in its audit, and "ran a 2,000-token `query` for something `grep` finds" too.

Without a map (Graphify not installed, or no `graphify-out/`), agents work as before: `grep` first, then read only the files that match.

## 3. What it installs and runs

Read this before installing, as for any plugin ([`CLAUDE_PLUGINS.md`](./CLAUDE_PLUGINS.md), "What to check in the repo").

| What | Where | Notes |
| --- | --- | --- |
| The `graphify` CLI | Your computer, through `uv` (Python 3.11.5–3.14) | The package is `graphifyy` (two y's). Other `graphify*` packages on PyPI aren't this project. |
| The `/graphify` skill | `.claude/skills/graphify/` (50 KB plus references) | Loaded only when invoked; see §1 for why agents use the CLI instead. |
| A `## graphify` section | Appended to `CLAUDE.md` | Its rule says to query before every first search. Our stricter cost rules (§2, `CLAUDE.md` working rules) win where they differ. |
| `.claude/CLAUDE.md` | New, three lines | Registers the `/graphify` trigger. |
| Hooks | `.claude/settings.json`: `PreToolUse` on `Bash`, `Grep`, `Read` and `Glob`; `SessionStart` after a compaction | Before a search or a file read, a reminder that the map exists (at most 5 a session). It blocks nothing unless strict mode is on (don't turn it on, §6). |
| The map | `graphify-out/` (`graph.json`, `GRAPH_REPORT.md`, `graph.html`, `manifest.json`, a cache) | Rebuilt locally; add it to `.gitignore`. |
| Git hooks (optional) | `.git/hooks/` | Rebuild the map after commit, checkout, merge and rebase. |
| A query log | `~/.cache/graphify-queries.log` | Questions and timings, not code. Off with `GRAPHIFY_QUERY_LOG_DISABLE=1`. |

**Network:** the code map never leaves the computer. Docs, PDFs and images need a model to read them: the `/graphify` skill uses the session's model, and `graphify extract` uses whichever API key it finds. This project maps code only, so neither happens. Don't use Graphify Cloud or its hosted MCP server without the owner's OK: they send the code to Graphify's servers.

## 4. Install (with the owner's OK)

The owner approves this like any dependency (`CLAUDE.md`), because it adds hooks and edits `CLAUDE.md`. Record the decision in `docs/ROADMAP.md` (owner decisions). From the project root:

```bash
uv tool install graphifyy            # once per computer (or: pipx install graphifyy)
graphify claude install --project    # this project only: skill, CLAUDE.md section, hooks
graphify update .                    # build the code map: local, no LLM, a few seconds
graphify hook install                # optional: rebuild after every commit and checkout
echo "graphify-out/" >> .gitignore
```

Check it: `graphify explain "ApiHealth()"` should list the Welcome page's API card and what it uses.

- **Project, not global.** `--project` keeps the skill and hooks in this repo. Plain `graphify install` puts the skill in your home folder for every project, so don't use it here.
- **Code only.** Build with `graphify update .`, not `/graphify .`: the skill reads docs with the model (tokens) and loads 13k tokens of instructions first. The docs don't need the map; `CLAUDE.md` says which doc answers what.
- **Commit** `.claude/skills/graphify/`, `.claude/CLAUDE.md`, `.claude/settings.json`, the `CLAUDE.md` section and the `.gitignore` line in one commit (`dksk/chore/add-graphify`), after `code-reviewer` has read the hook commands.

## 5. Keeping the map current

- **With git hooks** (`graphify hook install`): the map rebuilds after every commit, checkout, merge and rebase.
- **Without them:** run `graphify update .` after pulling or switching branches, and after a session that changed code. `graphify watch .` rebuilds on every save (needs `uv tool install "graphifyy[watch]"`).
- A stale map is worse than none: if an answer names a file or function that no longer exists, run `graphify update .` before going on.

## 6. Settings, limits and removal

- **Don't turn on strict mode** (`--strict` or `GRAPHIFY_HOOK_STRICT=1`). It blocks `Read` and `Grep` until the agent has queried the map, which forces a query even when `grep` or a known file is cheaper.
- **Skip folders** with a `.graphifyignore` (same format as `.gitignore`). Graphify already skips everything in `.gitignore`, so `node_modules/` and `dist/` stay out. Files it can't parse (such as `wrangler.jsonc`) are skipped with a note.
- **Removal:** `graphify claude uninstall --project`, `graphify hook uninstall`, then delete `graphify-out/` and the `.gitignore` line. Note it in `docs/ROADMAP.md`.
