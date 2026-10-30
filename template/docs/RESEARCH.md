# Research: competitors and UX patterns

A template for the research step that comes **before big UI work** (a new section, a new core flow, a redesign). `product-designer` runs it; `project-manager` adds it as its own task ahead of the design task. Small changes skip it. Save each round as `docs/research/YYYY-MM-DD-<topic>.md` and link it from the task in `ROADMAP.md`.

**Why:** people bring habits from the tools they already use (Jakob's law, `DESIGN_SYSTEM.md` §3.3). Learning what they expect is cheaper than redesigning after the owner sees a screen.

## 1. Scope (fill in first, 5 lines)

| | |
| --- | --- |
| Feature or flow | {{FEATURE}} |
| Who uses it and the job | {{ROLE}}: {{JOB}} |
| The decision this research informs | e.g. "list or board for the review queue" |
| Products to look at (3–6) | Direct competitors, plus 1–2 tools the users live in every day |
| Time box | e.g. one agent run; stop when the questions are answered |

## 2. Questions to answer

Answer each for every product; skip what doesn't apply.

| # | Question |
| --- | --- |
| 1 | What does the first screen show a new person, and an empty account? |
| 2 | How many steps from entry to the job done (the core loop)? Where are the decisions? |
| 3 | Where do the primary action, navigation, search, account and settings sit? Which keys work (⌘K, Esc, Enter)? |
| 4 | How are lists shown (rows, cards, table), filtered, sorted and paged? |
| 5 | What do loading, empty, error and success look like? What gets announced or undone? |
| 6 | How do roles and permissions show up (hidden, disabled with a reason, read-only)? |
| 7 | How do they build trust (sources, history, who changed what, previews before acting)? |
| 8 | What does it look like on a phone? |
| 9 | What do users praise or complain about (public reviews, forums, changelogs, help pages)? |
| 10 | What do they charge for, and what's free? (Only if pricing affects the feature.) |

Use public sources only: the product's own site, docs, help centre, changelog, public demos and reviews. Don't sign up with personal or company data unless the owner says so. Note the date and URL of each source.

## 3. Output format

Keep it to one page. Tables, not prose.

**Pattern table**

| Pattern | Seen in | Fits us? | Why (UX law or principle, e.g. §3.2 Hick) |
| --- | --- | --- | --- |
| e.g. Inline citations under each answer | A, C | Yes | Trust; §3.17 peak-end |

**Recommendations** (3–7, ranked): what to do, which pattern it borrows, which problem it solves, size (S/M/L).

**Quick wins** that fit the current milestone, and **bigger bets** for the roadmap, each as a candidate task for `project-manager`.

**Open questions for the owner.**

**Sources:** product, URL, date checked.

## 4. What not to copy

| Don't copy | Instead |
| --- | --- |
| Their brand: colours, logo, illustrations, fonts, wording | Our tokens and copy rules (`DESIGN_SYSTEM.md` §2, §20) |
| Marketing-site patterns in the app (hero banners, big gradients, pill buttons) | §1.1: a serious web app; buttons 10 px corners |
| Features nobody asked for, because a competitor has them | Only what serves the brief's job; the rest goes to "later" with the owner |
| Dark patterns: hidden cancel, forced sign-up, confirm-shaming, fake urgency | Honest copy (§1.9), easy exits |
| Patterns that fail WCAG 2.2 AA (hover-only actions, low-contrast text, tiny targets) | §17 and the §23 checklist |
| Screenshots or text pasted into our repo | Describe the pattern in words; link the source |
| Claims we can't back up ("AI that never makes mistakes") | Say what the app really does |

## 5. After the research

1. `product-designer` turns the recommendations into the brief and user flow (agent file, "Product thinking").
2. `project-manager` adds accepted items to `ROADMAP.md` (quick wins into the current milestone, bigger bets as a new one) and lists owner decisions.
3. Patterns adopted as rules go into `DESIGN_SYSTEM.md` §22.
