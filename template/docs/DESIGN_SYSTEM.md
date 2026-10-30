# {{PROJECT_NAME}} design system

The rules every screen follows. They were refined over many rounds of owner feedback on earlier apps built on this same stack, so start from them and change them deliberately. Tokens live in `src/index.css`; the building blocks named here live in `src/components/` and `src/lib/`. Where any other reference disagrees, **this file wins**.

Fill in the `{{…}}` placeholders, then record the owner's feedback for this project in §22 as it comes.

## Contents

1. [Principles](#1-principles)
2. [Colour](#2-colour)
3. [UX laws and how we apply them](#3-ux-laws-and-how-we-apply-them)
4. [Type](#4-type)
5. [Spacing](#5-spacing)
6. [Radius and elevation](#6-radius-and-elevation)
7. [Layout: the app shell](#7-layout-the-app-shell)
8. [Borders (one rule)](#8-borders-one-rule)
9. [Buttons](#9-buttons)
10. [Forms](#10-forms)
11. [Lists](#11-lists)
12. [Empty, loading and error states](#12-empty-loading-and-error-states)
13. [Toasts and announcements](#13-toasts-and-announcements)
14. [Dialogs, menus and overlays](#14-dialogs-menus-and-overlays)
15. [Motion](#15-motion)
16. [Route focus](#16-route-focus)
17. [Accessibility checklist (WCAG 2.2 AA)](#17-accessibility-checklist-wcag-22-aa)
18. [Responsive](#18-responsive)
19. [Dark mode](#19-dark-mode)
20. [Copy and voice](#20-copy-and-voice)
21. [Implementation notes (stack gotchas)](#21-implementation-notes-stack-gotchas)
22. [Project patterns and owner feedback](#22-project-patterns-and-owner-feedback)
23. [UX/UI review checklist](#23-uxui-review-checklist)

## 1. Principles

These are the owner's UI principles. Each has a one-line rule, how to apply it in this stack, a do and a don't, and what a reviewer checks. Cite them by number in critiques ("§1.2: two filled buttons in the header"). The review checklist in §23 collects every check.

### 1.1 A serious web app, not a marketing site
The app opens straight onto the first useful screen; no landing page, no hero.
- **Apply:** `/` is the screen people use daily. Brand moments (logo, colour) live in the shell, not in big banners.
- **Do:** open on the list or dashboard people came for. **Don't:** add a hero, a carousel or a "Welcome to {{PROJECT_NAME}}" splash.
- **Check:** after sign-in, the first screen has real content or a useful empty state with one action.

### 1.2 One brand colour, used with intent
The brand colour (`--primary`) fills the single main action in an area and draws focus rings; nothing else.
- **Apply:** `Button` default variant for the main action only; everything else is `outline`, `secondary` or `ghost`. Brand-coloured **text** always uses `text-primary-ink` (lighter in dark mode so it stays AA), never `text-primary`. Soft brand chips: `bg-primary/10 text-primary-ink`.
- **Do:** one filled brand button per area, `text-primary-ink` for links. **Don't:** two filled brand buttons side by side, brand colour as decoration, `text-primary` for text, a second accent colour.
- **Check:** count filled `bg-primary` buttons per area (at most one); `grep -rn "text-primary\b" src/` finds nothing outside `ui/`.

### 1.3 Red means destructive, nothing else
`--destructive` is only for delete, remove, sign out and their errors.
- **Apply:** `variant="destructive"` (soft) to start a destructive action; the confirming button in an `AlertDialog` repeats the action ("Delete item"). Destructive menu items use `variant="destructive"`, after a separator, at the end. Validation errors also use it.
- **Do:** red for "Delete…" and error text. **Don't:** red for "New" badges, emphasis, promotions or icons in neutral menus.
- **Check:** every red element is destructive or an error; every destructive action is confirmed or undoable (WCAG 3.3.4).

### 1.4 Flat, not heavy
Thin borders and soft tints; shadows only on things that truly float.
- **Apply:** surfaces use `border` and `bg-card`; feedback uses `bg-hover`, `bg-selected`, `bg-muted`. Shadows only from tokens: `shadow-card` (barely there), `shadow-bar` (scrolled top bar), `shadow-pop` (menus, popovers, dialogs, toasts). **Hover changes colour, not position.**
- **Do:** `hover:bg-hover` on a row, `hover:border-field-hover` on a control. **Don't:** `hover:-translate-y-1 hover:shadow-xl`, `shadow-2xl` on an inline card, a card inside a card inside a dialog.
- **Check:** `grep -rn "shadow-\[\|hover:-translate\|hover:scale" src/ --exclude-dir=ui` is empty (button press scale lives in `ui/button.tsx`).

### 1.5 The interface recedes so the content leads
Chrome is quiet; whitespace, type and surface changes do the work before borders and boxes.
- **Apply:** one stroke weight (`border`), generous space between groups (`mt-6`/`gap-6`), tight space inside them. Separate sections with space or a `SectionLabel` before adding a box. No decorative gradients; the canvas washes stay very faint.
- **Do:** let a list sit in one `ListCard` with dividers. **Don't:** box every block, add icons that repeat the label, or decorate empty space.
- **Check:** squint test: the main content and the one primary action are the first things you see; no nested cards.

### 1.6 Quiet, consistent type
Medium weight for emphasis, regular for reading; never bold UI.
- **Apply:** headings, labels and every button use `font-medium`; body is regular at `text-sm` (14 px) with line-height 1.55; long reading uses `text-reading`. Display sizes (`text-lg` and up) may tighten slightly (`tracking-[-0.01em]`); never tighten small text. `font-semibold` only for unread counts and similar badges.
- **Do:** `font-medium` page titles at `text-lg`. **Don't:** `font-bold` buttons, all-caps headings, more than three sizes on one screen.
- **Check:** `grep -rn "font-bold\|font-semibold" src/ --exclude-dir=ui` returns only badge/count uses.

### 1.7 Micro-interactions that respect reduced motion
Motion explains a change; it never decorates, and it is off with reduced motion.
- **Apply:** 150 ms feedback, 200 ms fades, 300 ms things moving into place; enter eases out, exit eases in; only `opacity` and `transform` animate. Every animation is `motion-safe:` (or covered by the safety net in `src/index.css`). Spinners and skeletons appear only after ~250 ms. Buttons press with a small scale (built into `Button`). Panels animate width so content never jumps.
- **Do:** fade a page in (`FADE`), settle list rows (`list-enter`). **Don't:** bounce, loop, parallax, or animate layout properties.
- **Check:** with `reducedMotion: "reduce"` nothing slides or scales and nothing stays hidden; no animation longer than 300 ms.

### 1.8 Accessible by default
WCAG 2.2 AA in both themes is the floor, not a phase.
- **Apply:** shadcn/Base UI primitives (they carry roles and states), visible `focus-visible` rings, labels on every control, announcements through `notify()` and `announce()`. See §17.
- **Do:** `aria-label` on icon-only buttons, real headings in order. **Don't:** `outline-none` without a ring, colour as the only signal, placeholders as labels.
- **Check:** zero axe violations; keyboard walk of the flow; contrast computed from tokens.

### 1.9 Plain, honest, short writing
Sentence case, short sentences, say what happened and what to do next.
- **Apply:** buttons are verbs (+ object); errors follow "what's wrong + how to fix it"; no em dashes in UI copy; never claim a feature that isn't built ("Coming soon" if needed). See §20.
- **Do:** "Couldn't save. Check your connection and try again." **Don't:** "Error 500", "Oops!", "Are you sure?", "Successfully deleted the item!".
- **Check:** read every string on the screen aloud; no codes, no blame, no hype, no em dashes.

### 1.10 Responsive from 320 to 1920 px
Every screen works on a 320 px phone and a 1920 px monitor.
- **Apply:** mobile first; `max-sm:` simplifies phones (labels to `sr-only`, stacked footers, bottom-sheet dialogs); container queries where a component's own width matters; text and controls scale up a little on very wide screens (already in `index.css`).
- **Do:** let rows wrap and titles clamp to two lines on phones. **Don't:** fixed widths, horizontal scrolling, hover-only controls on touch.
- **Check:** 320, 375, 768, 1280 and 1920 px: `scrollWidth <= clientWidth`, nothing overlaps, targets stay at least 24 px (44 px on touch for main actions).

### 1.11 Architecture rules
The structure follows the work, from broad to specific.
- **Apply:** navigation goes broad to specific, left to right (section → list → item). Quick tasks open as dialogs over the current context; bigger ones get their own page. Every screen has one primary action, and every multi-step flow has a way back. The global bar (search, theme, account) stays put while everything below it changes.
- **Do:** "Rename" in a dialog; "New project" with several steps on a page. **Don't:** a full page for a one-field edit, a wizard without Back, moving the global controls between pages.
- **Check:** the current section is marked (`aria-current="page"`); Back and Esc always lead somewhere sensible.

### 1.12 Design system first, and every change checked
Reuse before you build; check before you call it done.
- **Apply:** shadcn components and the shared building blocks (`Page`, `ListCard`, `ListRow`, `EmptyState`, `FormField`, `notify()`) before anything new; a new pattern gets written down here. Every change is checked at phone, tablet and desktop, light and dark, with no console errors.
- **Do:** extend a building block with a prop. **Don't:** a one-off list row or a hand-styled toast.
- **Check:** the §23 checklist passes; new patterns are recorded in §22.

## 2. Colour

All colours are tokens in `src/index.css` (`:root` for light, `.dark` for dark). Never hard-code a colour in a component; dark mode depends on the tokens. To rebrand, change the values marked `BRAND` and check contrast again.

### Brand and meaning

| Token | Use |
| --- | --- |
| `--primary` = {{BRAND_COLOR}} (default `#2563EB`) | The main action and selected states. **One primary button per area.** |
| `--primary-ink` | Brand-coloured text and links. Lighter in dark mode so it stays AA (lightness ~0.72). |
| `--accent` / `--accent-foreground` | A soft brand tint for highlighted (not selected) things. |
| `--success` | "Saved", "Copied" ticks. Green, AA on its own tint. |
| `--destructive` | Delete and errors only. |
| `--ring` | Focus rings (brand colour, `ring-ring/50`). |
| `--chart-1…5` | Charts only; `--chart-1` is the brand. |

Brand palette for brand content (logos, emails, exports), not UI chrome: {{BRAND_PALETTE}}.

### Neutrals and surfaces

| Token | Light | Use |
| --- | --- | --- |
| `--background` | white | The chrome: sidebar and top bar (**white**, never a dark nav). |
| `--canvas` + `canvas-surface` utility | a touch darker than white, two very faint brand washes | The working area behind cards. Keep the washes faint. |
| `--card` | white | Cards, lists, dialogs. |
| `--popover` | white | Menus, popovers, ⌘K, toasts. |
| `--foreground` | off-black with a hint of the brand hue | Body text. |
| `--muted-foreground` | ~5.5:1 on cards | Secondary text, meta lines, hints. |
| `--subtle-foreground` | ~3.9:1 | Decoration only (`·` separators, list markers). Never text that must be read. |
| `--muted` / `--secondary` | light grey | Quiet fills (chips, code, tags). |
| `--hover` / `--selected` | 4% / 6.5% of the ink | Row and nav feedback; work on any surface. |

Neutrals carry a faint tint of the brand hue (the third oklch number). If you change the brand hue, change it in the neutrals too.

### Strokes

| Token | Use |
| --- | --- |
| `--border` | Every surface stroke (see §8). |
| `--field` | Every form control border: soft and near-neutral by owner request. |
| `--field-hover` | Control hover; deepens towards the brand, never to near-black. |
| `--field-strong` | Checkboxes and radios, which are too small for the soft border. |
| `--input` | Dark-mode control fills only (`dark:bg-input/30`). Never a border. |

### Shadows

| Token | Class | Use |
| --- | --- | --- |
| `--elev-card` | `shadow-card` | Cards and lists barely lift. None in dark mode (the border does it). |
| `--elev-bar` | `shadow-bar` | The sticky top bar once the page scrolls. |
| `--elev-pop` | `shadow-pop` | Menus, popovers, dialogs, toasts, ⌘K: one floating shadow with the stroke built in. |

No `shadow-[…]` values in features; the button shadows in `button.tsx` are the one exception.

### Contrast

Compute ratios from the tokens for both themes, not from screenshots. Targets: text 4.5:1 (large text 3:1), UI parts and focus rings 3:1. Record the measured numbers here when the brand colour changes: {{CONTRAST_TABLE}}.

## 3. UX laws and how we apply them

Every design, build and review applies these. Each has a one-line definition, how to apply it in this stack, a do and a don't, and what a reviewer or QA looks for. Cite them in critiques ("Hick's law: two primary buttons in the header"). The checks are collected in §23.

### 3.1 Fitts's law
The time to reach a target depends on its size and distance: important targets are big and close.
- **Apply:** primary actions sit next to the content they act on (page header right, a sticky footer in long forms, the composer at the bottom of a chat) at `size="default"` (36 px) or `lg` (44 px). Every target is at least 24 × 24 px (WCAG 2.5.8); on touch, main actions are 40–44 px. Small icons get an `after:` hit area. Whole rows are clickable (`ListRow`).
- **Do:** make the full row the link. **Don't:** a 16 px icon as the only way to open an item, or the main action far from where the eye is.
- **Check:** bounding boxes of interactive elements are at least 24 px; main touch actions at least 40 px at 375 px wide.

### 3.2 Hick's law
More choices make decisions slower: one clear next step, the rest tucked away.
- **Apply:** one primary action per view; secondary actions are `outline` or `ghost`; rare or destructive ones go in a `…` menu. Long option lists get search or grouping. Forms ask only what's needed now.
- **Do:** "New item" as the only filled button; "Export", "Archive" in the `…` menu. **Don't:** five equal buttons in a header, or a settings page that shows every option at once.
- **Check:** count visible choices per area; one filled button; menus hold the rare actions.

### 3.3 Jakob's law
People expect your app to work like the others they use.
- **Apply:** familiar patterns: logo and nav top-left, left sidebar, ⌘K / Ctrl+K search, label above field with the submit button at the end, row click opens the item, Esc closes overlays, Enter submits single-line forms. Borrow from Linear, Notion, Slack, Google Workspace and iOS, and say which pattern you borrowed.
- **Do:** put the account and theme menus top-right. **Don't:** invent a new navigation model or move standard controls.
- **Check:** a first-time user can find nav, search, create and settings without help; Esc and Enter behave as expected.

### 3.4 Miller's law and chunking
People hold only a few things in mind at once: show small chunks.
- **Apply:** group fields into small titled sections (`fieldset` + `legend`, `SectionLabel`), 3–6 fields each; long lists get sections, filters or search; meta lines stay to 2–3 parts (`MetaLine`); one feature or idea at a time.
- **Do:** split a 15-field form into "Basics", "Access", "Details". **Don't:** one long wall of fields or a meta line with six parts.
- **Check:** no group has more than about 7 items without a heading or a filter.

### 3.5 Tesler's law
Complexity can't vanish, only move: the app carries it, not the person.
- **Apply:** smart defaults (today's date, the current user, the last choice); prefill what we already know (WCAG 3.3.7); tidy input as it's typed (trim, split pasted lists into chips); AI fill-in where it helps, with the person checking the result.
- **Do:** split "a@x.com, b@x.com" into two chips. **Don't:** ask for information the app already has.
- **Check:** no field asks for something already known in the flow; defaults are sensible.

### 3.6 Postel's law
Be liberal in what you accept and strict in what you send.
- **Apply:** accept flexible input (any case for email, commas, spaces or new lines between items, "1 000" or "1000", pasted values), normalise it, then validate kindly: errors only after submit, saying how to fix them. The server still validates strictly with zod.
- **Do:** trim spaces before validating. **Don't:** reject a pasted email because of a trailing space.
- **Check:** try odd but reasonable input (spaces, case, pasted lists); it's accepted or the error says how to fix it.

### 3.7 Doherty threshold
Feedback within about 400 ms keeps people in flow.
- **Apply:** press feedback at once; busy buttons disable at once and show a spinner after 250 ms; skeletons after 250 ms (`Delayed`); optimistic UI for safe writes (`setResource`, roll back with a toast on failure); old data stays on screen while it reloads (`useResource`); streaming for slow AI answers.
- **Do:** update the list immediately on create, then confirm. **Don't:** a blank screen or a frozen button while waiting.
- **Check:** with network throttling, every action shows a response within 400 ms; no layout jump when data arrives.

### 3.8 Progressive disclosure
Show what's needed now; reveal the rest on demand.
- **Apply:** optional fields and advanced settings stay hidden until asked for ("Add another…", a collapsible "More options"); rare actions in menus; details in a dialog or on the item's page; prompts appear in context (a limit explains itself when it's reached).
- **Do:** "More options" for the four rarely used fields. **Don't:** every option visible by default, or a feature tour before the first task.
- **Check:** the default view shows only what the main task needs; hidden options are findable in one step.

### 3.9 Law of proximity
Things close together look related.
- **Apply:** tighter spacing inside a group (`gap-2`) than between groups (`gap-6`, `mt-6` before a `SectionLabel`); a label sits closer to its field than to the field above; a hint sits right under its field.
- **Do:** label above, hint below, both tight to the input. **Don't:** equal spacing everywhere, or an action floating away from what it acts on.
- **Check:** in every form and list, gaps inside a group are smaller than gaps between groups.

### 3.10 Law of common region
A shared boundary makes a group.
- **Apply:** a card (`rounded-xl border bg-card`) holds one topic; a `ListCard` holds one list; a dialog holds one task. Don't nest cards: use inner tiles (`rounded-lg border`) or dividers.
- **Do:** one card per topic. **Don't:** a bordered card inside a bordered card inside a dialog.
- **Check:** no nested cards; each boxed region has one purpose.

### 3.11 Law of similarity
Things that look alike are read as working alike.
- **Apply:** the same kind of thing always has the same style: every list is a `ListCard`, every status a badge, every meta line a `MetaLine`, every row the same height and hover.
- **Do:** one badge style per status. **Don't:** a link styled like a button in one place and like text in another.
- **Check:** compare similar items across pages; they share component, size, colour and hover.

### 3.12 Uniform connectedness
Visually connected things are read as related.
- **Apply:** rows in a `ListCard` share one border with dividers; a form's actions share its card's footer; a breadcrumb links parent and child; a selected nav item and its page share the same name.
- **Do:** put a dialog's buttons in its footer bar. **Don't:** float form buttons outside the form's card.
- **Check:** actions sit inside the region they act on.

### 3.13 Von Restorff effect
The one thing that differs is remembered.
- **Apply:** exactly one filled brand button per area, red only for destructive, one badge or count that matters ("Reviews 3"). Everything else stays quiet so the difference means something.
- **Do:** a single primary button. **Don't:** brand colour on decoration, or several "highlighted" things competing.
- **Check:** squint test: the one important thing per area stands out, and only one.

### 3.14 Serial position effect
People remember the first and last items best.
- **Apply:** most important nav items first, the most used utility last; the primary action is the last button in a row (right-aligned); most recent or most relevant items first in lists.
- **Do:** Cancel then Save, Save on the right. **Don't:** bury the main section in the middle of the nav.
- **Check:** nav order follows importance; the primary action ends the button row.

### 3.15 Goal-gradient effect
People speed up as they get close to a goal.
- **Apply:** multi-step flows show progress ("Step 2 of 3", a `Progress` bar or numbered steps); count a step that's already done (creating the account) as step 1; keep each step short.
- **Do:** start setup at "Step 2 of 3" when the account already exists. **Don't:** hide how many steps are left.
- **Check:** every flow with more than one step shows where you are and how much is left.

### 3.16 Zeigarnik effect
Unfinished tasks stay on the mind.
- **Apply:** show unfinished work: drafts listed first, counts in the nav, "Continue where you left off", partly filled progress. Never lose work: guard unsaved changes (`useLeaveGuard`).
- **Do:** "2 drafts" at the top of the list. **Don't:** hide drafts in a separate place nobody visits.
- **Check:** leaving a half-done form asks first; drafts and pending items are visible where people start.

### 3.17 Peak-end rule
People judge an experience by its best moment and its end.
- **Apply:** end every flow well: a clear success state or toast ("Item created") with the next step or Undo; errors say what to do next; the empty state after the last item is friendly.
- **Do:** "Item deleted · Undo". **Don't:** end a flow on a blank page or a raw error.
- **Check:** walk each flow to its end; the last screen confirms what happened and offers a next step.

### 3.18 Aesthetic-usability effect
People find good-looking interfaces easier and forgive more.
- **Apply:** polish is part of the work: aligned edges, the spacing scale (§5), consistent radii (§6), tokens only, calm motion. A messy screen reads as a broken one.
- **Do:** align every edge in a card to the same padding. **Don't:** ship "roughly right" spacing to fix later.
- **Check:** edges align, spacing follows the scale, no stray sizes or colours.

### 3.19 Consistency
Identical actions look identical everywhere (WCAG 3.2.4).
- **Apply:** the same action has the same label, icon, variant and place everywhere: "New <thing>" uses the same `Plus` icon and wording in the sidebar, the list page and ⌘K; Delete is always the destructive variant behind a confirm; the same thing has the same name (keep a glossary in §20).
- **Do:** one word per concept ("Sign out" everywhere). **Don't:** "Remove" here, "Delete" there for the same action.
- **Check:** grep labels for the same action across pages; same icon, word and variant.

### Journeys
For each important flow, write a journey: each step, where it happens, how people likely feel (good, unsure, at risk) and what the design does about it. For every "at risk" step, name the specific design response (e.g. "Hits a limit → a friendly prompt about what an account adds, not an error"). Keep journeys in `GUIDE.md` next to the feature they belong to.

## 4. Type

- **Font:** {{FONT}} (default Helvetica Neue, `--font-sans` in `src/index.css`). A brand display font is for brand material, not app UI.
- **Weights:** `font-medium` for headings, labels and **all buttons, including primary**. Never `font-semibold` or `font-bold` on buttons (owner's rule). Body is normal weight.
- **Body:** 14 px (`text-sm`) in the UI, `line-height: 1.55`.

| Use | Class | Size / line height |
| --- | --- | --- |
| Page title (`h1`, one per page) | `text-lg font-medium tracking-[-0.01em]` (in `Page`) | 18 / 26 |
| Section title | `text-base font-medium`, optional 16 px Lucide icon in `text-primary-ink` | 16 / 24 |
| Section label | `SectionLabel`: `text-meta font-medium text-muted-foreground` | 13 / 18 |
| Body, list titles, buttons | `text-sm` | 14 / 20 |
| Meta, help, times | `text-meta text-muted-foreground` | 13 / 18 |
| Long reading text | `text-reading` | 15 / 26 |
| Document title | `text-doc font-medium` | 22 / 30 |
| Badges, tiny labels | `text-xs` | 12 / 16 |

- Small labels like "Step 1": muted, small, sentence case, never all caps and never brand-coloured.
- `text-meta`, `text-reading` and `text-doc` are registered with `cn` in `src/lib/utils.ts`. Import `cn` from `@/lib/utils`, never from the `cn` package directly, or `text-meta text-muted-foreground` loses its size.
- Numbers that change or line up use `tabular-nums`.
- Icons: Lucide, 16 px (`size-4`) beside text, stroke 1.75 (set globally). Decorative icons get `aria-hidden="true"`.

## 5. Spacing

Tailwind's 4 px scale. Use these steps, not arbitrary values.

| Step | Use |
| --- | --- |
| `gap-0.5` (2 px) | Nav items |
| `gap-1.5`–`gap-2` (6–8 px) | Icon and label, buttons in a row, chips |
| `gap-3`–`gap-4` (12–16 px) | Fields in a form, items in a card |
| `p-4` / `p-5 sm:p-6` | List rows / cards |
| `mt-6` + `mb-2` | Before and after a `SectionLabel` |
| `gap-6`–`gap-8` | Between groups and sections |
| `px-4 sm:px-6`, `pt-7 pb-16` | Page padding (`Page`) |

- Page width: `max-w-260` (wide, lists) or `max-w-3xl` (narrow, forms and reading), centred (`Page width`).
- Top bar and sidebar header are 52 px (`h-13`). Sidebar 15 rem.
- Rows: 44 px one-line, 56 px two-line (`ListRow`).

## 6. Radius and elevation

| Radius | px | Use |
| --- | --- | --- |
| `rounded-sm` | 4 | Kbd hints, tiny inline marks |
| `rounded-md` | 6 | Badges, chips, tags |
| `rounded-lg` | 10 | **Buttons, inputs, selects, nav items, inner tiles** |
| `rounded-xl` | 12 | Cards, lists, menus, popovers |
| `rounded-2xl` | 16 | Dialogs and sheets |

- **Buttons are 10 px. Never pills** (`rounded-full` only for avatars and dots).
- Cards: `rounded-xl border bg-card p-5 sm:p-6 shadow-card`. Inner tiles: `rounded-lg border`.
- Elevation comes only from the three shadow tokens (§2). Hover changes colour, never position.

## 7. Layout: the app shell

`src/components/app-shell.tsx`.

- **Sidebar** on wide screens (≥ 1024 px), 15 rem, white, with a 1 px right border. Nav items are 32 px high with an icon and a label; the active one is `bg-selected font-medium` with `aria-current="page"`. Icons tilt slightly on hover.
- **Phones and tablets:** the sidebar becomes a slide-in `Sheet` behind a menu button.
- **Top bar:** sticky, white at 85% with a background blur, gains `shadow-bar` once the page scrolls. Holds the menu button and logo on small screens, then (as the app grows) a breadcrumb, a centred search and the theme and profile menus on the right.
- **Canvas:** the working area uses `canvas-surface`, so white cards stand out.
- **Landmarks:** a "Skip to content" link, `aside` with a `nav`, `header`, `main#main`. One `LiveRegion` for announcements.
- **Sections:** listed in `NAV` (`src/lib/nav.ts`). Unbuilt sections may stay in the nav with an outline "Coming soon" badge and open an empty state.
- **Page frame:** `Page` (`src/components/page.tsx`) sets the width, the small `h1`, actions on the right and the tab title ("Overview · {{PROJECT_NAME}}").
- **Big screens:** the root font grows to 106.25% at 1920 px and 118.75% at 2400 px, so everything scales together.
- Optional, as the app grows: a collapsible sidebar (folds to 3.5 rem of icons with tooltips, remembered in localStorage) and ⌘K search (opens with ⌘K, Ctrl+K or `/` when not typing; every typed word must match; results show where they live).

## 8. Borders (one rule)

Two strokes, each with one job. Don't invent others.

| Token | Class | Use |
| --- | --- | --- |
| `--border` | `border` / `border-border`, `ring-border` | Every surface: cards, list rows and dividers, tables, sidebar and top bar edges, menus, popovers, dialogs, outline buttons |
| `--field` | `border-field` | Every form control: inputs, textareas, selects, OTP, input groups |

- Hover on a control: `hover:border-field-hover`. Never a near-black hover.
- Checkboxes and radios: `border-field-strong`.
- Focus is the ring, not a different border colour: `focus-visible:border-ring focus-visible:ring-3 focus-visible:ring-ring/50` (`focus-within:` / `has-[…:focus-visible]:` for wrappers). A control that's focused almost all the time (a chat composer) may use a softer halo: `border-ring ring-4 ring-ring/12`.
- Brand-tinted borders (`border-primary/…`) only for selected or highlighted states, never as a plain container edge.
- Allowed exceptions: status badge tints where colour supports the word, a destructive error card (`border-destructive/30`), and one dashed "nothing here" style if the app needs it. Document any other exception here.

## 9. Buttons

`src/components/ui/button.tsx`.

- 10 px corners, `font-medium`. Primary has an inset highlight and darkens on hover; outline has a hairline shadow; a quick press to `scale(0.98)` (150 ms) that is off with reduced motion.
- Variants: `default` (primary; **one per area**), `outline` (secondary), `ghost` (quiet and icon buttons), `link`, `destructive` (delete only, behind a confirm).
- Sizes: `sm` in dense rows and toolbars, `default` most places, `lg` for the main action on touch-first screens; `icon` / `icon-sm` for icon-only buttons, which always have `aria-label` (and a tooltip when the icon isn't obvious).
- **Buttons are verbs:** "Save", "Create item", "Download". Not "OK" or "Submit".
- An icon goes before the label, 16 px, `aria-hidden`.
- **Links that look like buttons** use `<Button nativeButton={false} render={<a href=… onClick={onLinkClick} />}>`, so screen readers say "link". For raw classes, `cn(buttonVariants({ variant, size }))`.
- **Busy:** disable the button and show a `Spinner` (`ui/spinner.tsx`) only after ~250 ms, so quick actions don't flicker; keep the label ("Saving…"). If the app needs this often, add it to `Button` as a `loading` prop.
- Rare actions get a muted look (ghost or link) or live in a `…` menu, not in the primary colour.
- Copy buttons (`CopyButton`) swap to a green tick and "Copied" for 1.6 s and announce it.

## 10. Forms

`src/components/form-field.tsx`, `src/components/form-select.tsx`, `src/components/ui/field.tsx`.

- Every input has a **visible label** above it (placeholders aren't labels). Optional fields say "(optional)" in the label; required is the default.
- Controls are 40 px high with 10 px corners (`FormSelect`, `Input`), `border-field`.
- Help text sits under the field (`FieldDescription`) and is tied with `aria-describedby`.
- **Errors appear only after a submit attempt**, never for an empty field someone just left. They replace the help text, are tied with `aria-describedby` and `aria-invalid`, and focus moves to the first one. They say what's wrong and how to fix it ("Use a file under 10 MB").
- A message about the whole form (the server's answer) is a `FormAlert` above the submit button.
- Group related fields (`fieldset` + `legend`, 3–6 fields per group). Add `min-w-0` to fieldsets.
- Use `autocomplete` on personal data and allow paste and password managers (WCAG 3.3.8). Passwords get a Show toggle (`FormField password`).
- Long forms keep their actions in a sticky footer inside the card. Primary last, on the right.
- **Unsaved work:** `useLeaveGuard(dirty, message)` (`src/lib/route.ts`) asks before leaving. A stale save (409) says someone else changed it and offers Reload.
- Native `select` (`FormSelect`) for simple choices: it works best on phones and with screen readers. `Combobox` only when people need to search the options.

## 11. Lists

`src/components/page.tsx`.

- **`ListCard`:** every list is one bordered card with divided rows (`ul`, `aria-label`). Rows settle in with a short stagger the first time (`.list-enter`).
- **`ListRow`:** a 14 px `font-medium` title that is a link, and **the whole row is the target** (an `after:` overlay) with the focus ring drawn inside it. One meta line under the title. A trailing slot for a status badge or a quiet hint; buttons in the row (`actions`) sit above the link with their own target. On phones the trailing slot moves under the meta and the title can wrap to two lines.
- **`MetaLine`:** 2–3 parts separated by `·` ("Kari Nordmann · Updated today"). Each dot is decoration (CSS, silent for screen readers), and a wrapped line never starts or ends with a dot.
- **Order in a row:** status (if it drives action) or title first, then one meta line. Dates are short and relative ("Updated today", "Updated 2 Oct").
- **Status badges:** outline badges where the word carries the meaning; colour only supports it (muted Draft, brand-tint In review, success-tint Published, red outline for problems).
- **`SectionLabel`** above a group of rows ("Due this week").
- Lists that could get long get search, filters or sections; lists inside a page that could grow go in a scroll box or popover.
- **Wide tables on phones:** turn each row into a small card below 640 px instead of scrolling sideways.

## 12. Empty, loading and error states

`src/components/states.tsx`. Every screen has all three, plus success.

- **Empty (`EmptyState`):** an icon, a title, one line, and an action only if there's a real one ("New item"). `variant="page"` when it's the only thing on the page; `variant="inset"` inside a list's place (a filter with no matches).
- **Loading:** skeletons **shaped like the content** (`ListSkeleton` for lists), inside `Delayed`, which keeps them invisible for 250 ms so quick loads never flash. Old data stays on screen while it reloads (`useResource`, `useLatest`). Never a full-page spinner.
- **Error (`ErrorState`):** `role="alert"`, what happened, the server's message, and Retry.
- **No access (`NoAccessPage`)** and **not found (`NotFoundPage`)**: say so plainly and offer the way back.
- **Stale app after a deploy:** `ChunkBoundary` offers Reload instead of a blank page.

## 13. Toasts and announcements

- **`notify(title, { description, type, action, onClose })`** (`src/lib/notify.ts`) for completed actions ("Saved", "Item deleted") and failures. `notifyError(title, error)` uses the server's message.
- A toast can carry **one action** (Undo). Work that waits for Undo runs in `onClose`, so it respects the toast's paused timer (WCAG 2.2.1).
- Toasts never get the dialog role; errors get `role="alert"`. They sit clear of sticky action bars so they never cover Save.
- **`announce(text)`** (`src/components/live-region.tsx`): short confirmations ("Copied", "3 results") through the one app-wide polite live region, not a status element per button.

## 14. Dialogs, menus and overlays

- **Dialogs** (`dialog.tsx`, `alert-dialog.tsx`): `rounded-2xl`, `shadow-pop`, a clear title, one primary action last. Use `AlertDialog` to confirm destructive actions ("Delete item?" with the item's name, and a destructive "Delete" button). On phones a dialog may become a bottom sheet.
- Focus moves into the dialog and returns to the trigger on close; Base UI's `finalFocus` decides, so return `false` when your action moves focus itself.
- **Menus and popovers:** `rounded-xl`, `shadow-pop`, open in 150 ms. Destructive items are last, after a separator.
- **Tooltips** name icon-only buttons and folded nav; never hide essential information in one.
- **Sheets** for the phone nav and side panels.
- Overlays animate with Base UI's `data-open:` / `data-closed:` (menus, popovers, dialogs) and `data-starting-style:` / `data-ending-style:` (sheets, toasts). Esc always closes.

## 15. Motion

Motion explains a change; it never decorates. Tokens live in `src/index.css`, helpers in `src/lib/motion.ts`.

| Token | Value | Use |
| --- | --- | --- |
| `duration-150` | 150 ms | Hover, press, tabs, menus and popovers opening |
| `duration-200` | 200 ms | Fades (page change, tab panel, skeleton to content), exits, backdrops |
| `duration-300` | 300 ms | Things that move into place: dialogs, sheets, toasts, list rows. **Nothing is longer.** |
| `ease-enter` | `cubic-bezier(0.32, 0.72, 0, 1)` | Everything that appears (fast start, soft landing). `ease-ios` is the same curve. |
| `ease-exit` | `cubic-bezier(0.4, 0, 1, 1)` | Everything that leaves, a little faster than its entrance |
| `animate-fade` / `FADE` | opacity, 200 ms | A page after a route change (`AppShell`), a new tab panel |
| `animate-enter` / `ENTER` | opacity + 4 px rise, 300 ms | Cards and tiles; `enterDelay(i)` staggers 30 ms apart, capped at 150 ms |
| `animate-enter-up` | opacity + 8 px rise, 300 ms | Content appearing below the reader's eye (a new message, a sticky bar) |
| `.list-enter` | `animate-enter` per child, 30 ms stagger, max 150 ms | Rows of every `ListCard` when they first appear |

Rules:

- Only `opacity` and `transform` animate. Colour transitions are fine for hover. One layout exception is allowed: a sidebar folding (`grid-template-columns`, 300 ms).
- Buttons press to `scale(0.98)` for 150 ms (not menu triggers).
- Skeletons wait 250 ms, then fade in; content replaces them with the list stagger, so nothing jumps.
- History never animates: only things that are new to the person move.
- **Reduced motion:** every animation is `motion-safe:` (or inside `@media (prefers-reduced-motion: no-preference)`), and a global safety net in `src/index.css` sets durations to ~0 under `prefers-reduced-motion: reduce`, so third-party primitives stop moving too. Delays are kept, so `Delayed` still hides quick skeletons. Colour changes still happen, instantly.
- Small touches are fine: nav icons tilt slightly on hover, the theme icon turns on hover.
- Drag, layout and shared-element transitions that CSS can't do use Motion; decorative effects (3D, glass, moving gradients, liquid metal, animated backgrounds, smooth scroll, scroll-scrubbed timelines) are owner opt-in exceptions to this section, never in the app shell. Both follow `DESIGN_INSPIRATION.md`.

## 16. Route focus

`src/lib/route-focus.ts`, used by `AppShell`.

- After an in-app navigation, if focus was lost (the clicked link is gone), focus moves to the new page's `h1` (`tabindex=-1`, no ring) and follows it while a loading heading is replaced (WCAG 2.4.3).
- If something else already has focus (a text box, a sidebar link that's still there), it stays, and the page title is announced instead.
- Never on the first load.
- Each page sets a unique tab title through `Page` / `useDocumentTitle` ("Overview · {{PROJECT_NAME}}", WCAG 2.4.2).
- `scroll-padding-top` keeps focused elements clear of the sticky top bar (WCAG 2.4.11).

## 17. Accessibility checklist (WCAG 2.2 AA)

The full rules, with criterion numbers, are in `.claude/agents/product-designer.md`. Before calling a screen done:

- [ ] **Contrast:** text 4.5:1 (large 3:1), UI parts, borders of controls and focus rings 3:1, in both themes (1.4.3, 1.4.11).
- [ ] **Colour is never the only signal:** errors, status and selection also have text or an icon (1.4.1).
- [ ] **Structure:** one `h1`, headings in order, real lists, tables with headers, labels tied to inputs, `fieldset`/`legend` for groups (1.3.1).
- [ ] **Keyboard:** everything works with Tab, Shift+Tab, Enter, Space, Esc and arrows; no traps; order follows the layout (2.1.1, 2.1.2, 2.4.3).
- [ ] **Focus visible** on everything, never hidden behind the sticky bar or a toast (2.4.7, 2.4.11).
- [ ] **Names:** icon-only buttons have `aria-label`; the accessible name contains the visible label (2.5.3, 4.1.2).
- [ ] **Target size:** at least 24 × 24 px; 40–44 px on touch (2.5.8).
- [ ] **Skip link and landmarks** (2.4.1); a unique page title (2.4.2).
- [ ] **Status messages** announced without moving focus (`role="status"`, `announce()`) (4.1.3).
- [ ] **Errors:** in text, next to the field, announced, with how to fix (3.3.1, 3.3.3).
- [ ] **Hover and focus content** is dismissible with Esc, hoverable and persistent (1.4.13).
- [ ] **Dragging** always has a non-drag alternative (2.5.7).
- [ ] **Sign-in** allows paste and password managers; no puzzles (3.3.8).
- [ ] **No re-typing** of information already given in the same flow (3.3.7).
- [ ] **Reflow:** nothing scrolls sideways at 320 px; usable at 200% zoom (1.4.10, 1.4.4).
- [ ] **Reduced motion** respected; nothing flashes (2.3.1).
- [ ] **Links that open a new tab** use `target="_blank" rel="noopener noreferrer"` and say so.
- [ ] **Hover-only controls** also appear on keyboard focus (`group-focus-within:opacity-100`) and are always visible on touch.
- [ ] **Busy and loading:** busy buttons set `aria-busy`; skeletons are labelled ("Loading…" in `sr-only`).
- [ ] **Decorative** icons and images are `aria-hidden="true"` / `alt=""`; hidden panels use `inert`.
- [ ] **Single-key shortcuts** never fire while typing in a field (2.1.4).
- [ ] **Inputs are 16 px on phones** so iOS doesn't zoom; bottom sheets respect `env(safe-area-inset-bottom)`.
- [ ] **Everything clickable** shows the pointer cursor (base-layer rule in `index.css`).
- [ ] **Zero axe violations** on the page.

## 18. Responsive

- Check **320, 375, 768, 1024, 1280/1440 and 1920 px**. Nothing may scroll sideways; `main` uses `overflow-x-clip`.
- Breakpoints: `sm` 640, `md` 768, `lg` 1024 (sidebar appears), `xl` 1280, `2xl` 1536, `3xl` 1920.
- Watch for: fieldsets need `min-w-0`; grids use `minmax(0,1fr)` columns; header columns use `minmax(max-content,1fr)` so the logo never squeezes; long words wrap (`wrap-break-word`); truncated titles get a `title`.
- Phones: primary actions within thumb reach (sticky bottom bar for long forms), 40–44 px targets, the nav in a sheet.
- Works in portrait and landscape (1.3.4).

## 19. Dark mode

- Light, dark and system, chosen in the theme menu (and with the `d` key when not typing), stored in localStorage as `theme`. A small script in `index.html` applies it before first paint, so there's no flash.
- Dark layers step up so each is visible: canvas 0.18 < chrome 0.205 < card 0.228 < popover 0.255 (oklch lightness).
- The border is an alpha white (`oklch(1 0 0 / 0.08)`) so one stroke works on every layer; card shadows are off (the border does the work).
- `--primary-ink` and `--ring` are lighter in dark mode so brand text and focus stay AA.
- Brand content on a backdrop (logos, exports) sits on a fixed light or dark colour, not a theme token, or dark logos vanish in dark mode.
- Every check in §17 runs in both themes.

## 20. Copy and voice

### Tone
- **Plain:** everyday words ("Log in", not "Authenticate").
- **Honest:** say what happens, including limits and consequences ("This can't be undone. Here's what happens:"). Don't claim what the app can't do; "Coming soon" for unbuilt things.
- **Short:** one idea per sentence; descriptions are one sentence. Take text out rather than add it; no descriptive lines under page titles unless they earn their place.
- **Friendly, not cute:** no exclamation-mark hype, no "Oops!".
- **Second person:** talk to "you"; the app says "we" rarely.

### Mechanics
- **Sentence case** everywhere: titles, buttons, menu items, labels ("New item", not "New Item").
- **No em dashes in UI copy.** Use a period, comma, colon or middle dot.
- **Middle dot (·)** joins related bits on one line: "Step 2 of 3 · Details", "Enter to send · Shift + Enter for a new line".
- **Ellipsis (…)** for progress ("Saving…"), menu items that open a confirmation ("Delete item…") and live activity ("Uploading…").
- **Numbers** as digits, pluralised correctly ("1 item", "3 items"), counts capped at "99+".
- **Keys:** `⌘K` on Mac, `Ctrl+K` elsewhere; "Enter", "Esc", "Shift + Enter".
- **Dates** short and relative in lists ("Updated today", "2 Oct"); full dates in details.
- **Placeholders** show an example format, never the label.

### Buttons and verbs
Buttons say exactly what happens, as a verb (+ object). The confirming button repeats the action from the dialog title ("Delete item?" → **Delete item**). Loading labels use the "-ing…" form of the same verb ("Saving…", "Creating…"). Links say where they go (no "Click here", no bare "More").

### Error message formula
**What's wrong (or what to do) + how to fix it, with an example when useful.** Never blame, never show codes.

| Pattern | Example |
| --- | --- |
| Empty required field | "Enter a project name, like your team or client." · "Enter your email, like name@company.com." |
| Too long / too short | "Use 40 characters or fewer." · "Use at least 8 characters." |
| Already taken | "There's already a project called “Apollo”. Try a different name." |
| Wrong credentials | "That email and password don't match. Check them, or reset your password." |
| Format rule | "Use only lowercase letters, numbers and dashes." |
| Rate limit | "Too many tries. Wait 15 minutes and try again." (use the server's `Retry-After`) |
| Unknown / network | "Something went wrong. Check your connection and try again." |
| Expired link | "This link has expired or was already used. Ask for a new one." |
| Action failed (toast) | "Couldn't copy" + "Select the text and copy it instead." |

Where errors go: field errors replace the field's hint; form-level problems go in a `FormAlert` above the submit button; failed actions are error toasts (`notify(…, { type: "error" })`).

### Hints
Every field that isn't obvious has a hint saying why it's asked or what's expected ("You'll log in with this.", "At least 8 characters.").

### Empty states
Say what will appear here and how to start, with one action if there's a real one ("No items yet. Items you create show up here." + **New item**).

### Say / not

| Say | Not |
| --- | --- |
| Couldn't save. Check your connection and try again. | Error 500 |
| Item deleted | Successfully deleted the item! |
| It's removed for everyone, along with its comments. | Are you sure? |
| No items match “zed”. | No results found |
| Reconnecting… Changes will save when you're back. | Connection error |

### Glossary
Keep one word per concept and record it here (§3.19): {{GLOSSARY}} (e.g. "Sign out", never "Log out" for the same action).

### Screen-reader copy
Write labels as full phrases: "Back to items", "Remove ada@company.com", "Show password", "Reviews, 3 waiting". Hidden context uses `sr-only` (", 3 unread").

## 21. Implementation notes (stack gotchas)

- shadcn `base-nova` is built on **Base UI**: use the `render` prop, not `asChild`.
- **Base UI groups** take their name from their last label, so use one label per group.
- **Combobox:** ignore empty `onInputValueChange` values unless `reason === "input-change"`, or Escape wipes what was typed.
- **Dialogs:** `finalFocus` decides where focus goes on close; return `false` when your action moves focus itself.
- **A status inside a button** drops the button's name; put the announcement next to it (or use `announce()`).
- **localStorage:** wrap every read and write in try/catch. Use `useSyncExternalStore` for stores that several components share.
- **Data:** `useResource(path)` shares one cached GET per path; after a write, `setResource`/`updateResource` or `invalidate("/api/…")`.
- **Keep files:** never delete shadcn components or brand files, even unused ones (owner's rule).

## 22. Project patterns and owner feedback

Record patterns specific to this app and the owner's feedback here, newest first, so every agent follows them.

| Date | Feedback or pattern |
| --- | --- |
| {{DATE}} | {{FEEDBACK}} |

Standing feedback carried over from earlier projects:

- The top bar and sidebar are **white**. A dark nav was tried and rejected.
- The canvas washes must be **very subtle**.
- Form control borders are **soft** (`--field`), not a harsh 3:1 grey; labels, the white field and the focus ring make fields identifiable.
- Buttons: 10 px corners, `font-medium`, never pills.
- Downloads: choose a file type, then press Download; never download straight from a menu item.
- Rare or destructive management actions live out of the way (a `…` menu or the profile menu), not inline.
- Optional extras stay hidden until needed.

## 23. UX/UI review checklist

The one checklist `product-designer` (before handing a design over), `developer` (before handing a build over), `ux-reviewer` (auditing a screen) and `qa-tester` (in the browser) all use. Report a failure with its number, where it fails, who it hurts, and the fix (file + change). Text checks first; screenshots only as JPEG (quality 60, CSS scale), never PNG.

**Structure and flow**
- [ ] The first screen is useful: real content or a helpful empty state (§1.1).
- [ ] One primary (filled brand) button per area; rare actions in a `…` menu (§1.2, §3.2, §3.13).
- [ ] Quick tasks in dialogs, bigger ones on pages; every multi-step flow has Back and shows progress (§1.11, §3.15).
- [ ] Optional and advanced options are hidden until asked for, and findable in one step (§3.8).
- [ ] Groups have at most about 7 items without a heading, filter or search (§3.4).
- [ ] Nothing asks for information the app already has; sensible defaults (§3.5).
- [ ] Unsaved work is guarded; drafts and pending items are visible where people start (§3.16).
- [ ] The flow ends on a clear confirmation with a next step or Undo (§3.17).

**Familiar and consistent**
- [ ] Standard placements: nav left, account and theme top-right, search on ⌘K / Ctrl+K, Esc closes, Enter submits (§3.3).
- [ ] The same action has the same label, icon, variant and place on every page (§3.19).
- [ ] Similar things share one component and style (lists, rows, badges, meta lines) (§3.11).

**Layout and visuals**
- [ ] Space inside a group is tighter than between groups; labels and hints hug their field (§3.9).
- [ ] One card per topic; no nested cards; actions sit inside the region they act on (§3.10, §3.12).
- [ ] Flat surfaces: tokens only, no `shadow-[…]`, hover changes colour not position (§1.4).
- [ ] Brand text uses `text-primary-ink`; red only for destructive and errors (§1.2, §1.3).
- [ ] Type: `font-medium` for headings, labels and buttons; no bold UI; at most three sizes per screen (§1.6).
- [ ] Edges align, spacing follows the scale, radii follow §6 (buttons and inputs 10 px, never pills) (§3.18).

**Feedback and states**
- [ ] Every action responds within 400 ms (press feedback, busy button, optimistic update) (§3.7).
- [ ] Loading shows a layout-shaped skeleton after 250 ms; old data stays while reloading (§12).
- [ ] Empty, error (what happened + what to do + Retry) and success states exist (§12).
- [ ] Destructive actions are confirmed or undoable (§1.3, WCAG 3.3.4).
- [ ] Status changes are announced (`notify()`, `announce()`) without moving focus (§13).

**Targets, input and responsive**
- [ ] Targets at least 24 × 24 px; main touch actions 40–44 px (§3.1).
- [ ] Flexible input is accepted and normalised; errors appear after submit and say how to fix (§3.6, §10).
- [ ] Works at 320, 375, 768, 1280 and 1920 px with no horizontal scroll or overlap (§1.10, §18).
- [ ] Inputs are 16 px on phones (no iOS zoom); dialogs become bottom sheets on phones (§10, §14).

**Accessibility (WCAG 2.2 AA)**
- [ ] Zero axe violations, in light and dark (§17).
- [ ] Keyboard only: logical Tab order, visible focus everywhere, no traps, Esc closes, focus returns to the trigger; after navigation focus lands on the new page's `h1` (§16).
- [ ] One `h1` per page; landmarks (`header`, `nav`, `main`); a skip link (§7, §17).
- [ ] Every control has a name: labels on inputs, `aria-label` on icon-only buttons; errors tied with `aria-describedby` (§17).
- [ ] Contrast from the tokens: text 4.5:1, UI parts and focus rings 3:1, in both themes (§2).
- [ ] Colour is never the only signal (§17).

**Motion**
- [ ] Only `opacity` and `transform` animate, 150/200/300 ms, `motion-safe:` (§1.7, §15).
- [ ] With reduced motion nothing slides or scales and nothing stays hidden (§15).

**Copy**
- [ ] Sentence case, verbs on buttons, no em dashes, no codes or blame, nothing claimed that isn't built (§1.9, §20).
- [ ] Errors: what's wrong + how to fix it, with an example when useful (§20).

**Done means**
- [ ] Checked at phone, tablet and desktop, light and dark, with no console errors (§1.12).
