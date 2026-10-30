# Testing

What to test and how: Worker tests in the real Workers runtime with an in-memory D1 and R2, front-end checks, browser QA with text-based checks first, and accessibility. The quality bar: every build task passes `pnpm typecheck`, `pnpm lint`, `pnpm test` and `pnpm build`, then `code-reviewer`, then (for UI) `ux-reviewer` and `qa-tester` ([`WORKFLOW.md`](./WORKFLOW.md)).

## Contents
1. [The Worker test harness](#1-the-worker-test-harness)
2. [What to test on the server](#2-what-to-test-on-the-server)
3. [Migration tests](#3-migration-tests)
4. [Front-end checks](#4-front-end-checks)
5. [Browser QA with Playwright (text first)](#5-browser-qa-with-playwright-text-first)
6. [Accessibility (axe)](#6-accessibility-axe)
7. [Before calling it done](#7-before-calling-it-done)

## 1. The Worker test harness

Tests run inside `workerd` through `@cloudflare/vitest-plugin`, with the bindings from `wrangler.jsonc` backed by **local in-memory D1, R2 and KV**. Migrations are read in Node (config time) and applied at the start of each test file. Storage is reset between test files by the plugin.

```ts
// vitest.config.ts
import { cloudflareTest, readD1Migrations } from "@cloudflare/vitest-plugin"
import { defineConfig } from "vitest/config"

export default defineConfig({
  plugins: [
    cloudflareTest(async () => ({
      wrangler: { configPath: "./wrangler.jsonc" },
      miniflare: {
        // Everything the API reads is set here, so tests never depend on a local
        // .dev.vars (gitignored; these values override it).
        bindings: {
          ENVIRONMENT: "test",
          DEMO_MODE: "false",
          SIGNUP_ENABLED: "true",
          ANTHROPIC_API_KEY: "",
          TEST_MIGRATIONS: await readD1Migrations("./migrations"),
        },
      },
    })),
  ],
  test: {
    include: ["worker/**/*.test.ts"],
    fsModuleCache: false, // workerd can't read vitest's on-disk module cache
    testTimeout: 30_000,
    hookTimeout: 60_000,
  },
})
```

```ts
// worker/test/harness.ts
import { applyD1Migrations, env as bindings } from "cloudflare:test"
import type { D1Migration } from "cloudflare:test"

// Tests name the caller with X-Test-User (honoured only when ENVIRONMENT is "test");
// the first one to call becomes the admin.
export const TEST_ADMIN = "first.admin@example.com"
// Requests that change something need a same-origin Origin (the CSRF check).
export const TEST_ORIGIN = "https://app.test"

export async function startTestEnv() {
  const { TEST_MIGRATIONS, ...rest } = bindings as Env & { TEST_MIGRATIONS: D1Migration[] }
  const env: Env = { ...rest, ENVIRONMENT: "test", ANTHROPIC_API_KEY: "" }
  await applyD1Migrations(env.DB, TEST_MIGRATIONS)
  return { env }
}
```

```ts
// worker/test/api.ts: one helper every test uses
import app from "../index"
import { TEST_ORIGIN } from "./harness"

export type Res<T> = { status: number; body: T; headers: Headers }

export async function api<T>(env: Env, user: string | null, method: string, path: string, body?: unknown): Promise<Res<T>> {
  const headers: Record<string, string> = { Origin: TEST_ORIGIN }
  if (user) headers["X-Test-User"] = user
  let payload: BodyInit | undefined
  if (body instanceof FormData) {
    // Serialise like a browser does, with a Content-Length (uploads require it).
    const encoded = new Request("https://app.test/", { method: "POST", body })
    payload = await encoded.arrayBuffer()
    headers["Content-Type"] = encoded.headers.get("Content-Type")!
    headers["Content-Length"] = String(payload.byteLength)
  } else if (body !== undefined) {
    headers["Content-Type"] = "application/json"
    payload = JSON.stringify(body)
  }
  const res = await app.fetch(new Request(`https://app.test/api${path}`, { method, headers, body: payload }), env)
  const parsed = res.headers.get("Content-Type")?.startsWith("application/json") ? await res.json() : new Uint8Array(await res.arrayBuffer())
  return { status: res.status, body: parsed as T, headers: res.headers }
}
```

```ts
// worker/test/raw.d.ts: lets tests import SQL files (seeds) as strings with ?raw
declare module "*?raw" {
  const content: string
  export default content
}
```

The test-user header is the **only** test hook in production code, and it's ignored unless `ENVIRONMENT` is `test` (assert that in a test too). AI calls use a stub client set with `setTestClaudeClient` ([`AI_FEATURES.md` §9](./AI_FEATURES.md#9-testing-ai-features)).

## 2. What to test on the server

Each endpoint gets a happy path **and** the refusals. Tests in one file share a database and run in order, so name users per test and don't depend on another file.

```ts
import { beforeAll, describe, expect, it } from "vitest"
import { startTestEnv, TEST_ADMIN } from "./harness"
import { api } from "./api"
import app from "../index"

let env: Env
const OWNER = "olivia.owner@example.com"
const OTHER = "oscar.other@example.com"

beforeAll(async () => {
  ;({ env } = await startTestEnv())
  await api(env, TEST_ADMIN, "GET", "/me") // first caller becomes admin
})

describe("items", () => {
  it("lets the owner rename, and refuses everyone else (IDOR)", async () => {
    const created = await api<{ id: string; revision: number }>(env, OWNER, "POST", "/items", { title: "Plan" })
    expect(created.status).toBe(201)
    const other = await api(env, OTHER, "PATCH", `/items/${created.body.id}`, { title: "Mine now", revision: 1 })
    expect(other.status).toBe(404) // not 403: don't reveal it exists
    const own = await api(env, OWNER, "PATCH", `/items/${created.body.id}`, { title: "Plan B", revision: 1 })
    expect(own.status).toBe(200)
  })

  it("returns 409 for a stale revision (race)", async () => {
    const { body } = await api<{ id: string }>(env, OWNER, "POST", "/items", { title: "Race" })
    const [a, b] = await Promise.all([
      api(env, OWNER, "PATCH", `/items/${body.id}`, { title: "A", revision: 1 }),
      api(env, OWNER, "PATCH", `/items/${body.id}`, { title: "B", revision: 1 }),
    ])
    expect([a.status, b.status].sort()).toEqual([200, 409])
  })

  it("refuses cross-site writes (CSRF)", async () => {
    const res = await app.fetch(
      new Request("https://app.test/api/items", { method: "POST", headers: { Origin: "https://evil.test", "X-Test-User": OWNER, "Content-Type": "application/json" }, body: "{}" }),
      env,
    )
    expect(res.status).toBe(403)
  })

  it("rate-limits with Retry-After", async () => {
    let last
    for (let i = 0; i < 11; i++) last = await api(env, null, "POST", "/auth/login", { email: "x@example.com", password: "wrong-password" })
    expect(last!.status).toBe(429)
    expect(Number(last!.headers.get("Retry-After"))).toBeGreaterThan(0)
  })
})
```

**Checklist per change**

| Area | Tests |
| --- | --- |
| Authn | Signed out → 401; expired session → 401 and cookie cleared; temporary password → only the allowed routes; the test header is ignored outside `test` |
| Authz | Each role that must be refused (403); another person's id (404, IDOR); visibility in lists, search, counts, downloads, shared links and AI requests |
| Validation | Missing, too long, wrong type → 400 naming the field |
| Races | Parallel writes with the same revision → exactly one 200 and one 409; single-statement limits hold under `Promise.all` |
| Limits | 429 with `Retry-After`; refunds where promised |
| Files | Too big (413), no `Content-Length` (411), wrong magic bytes (400), sanitised names, download headers, permission per download |
| Tokens | Token works only on its routes; revoked on password change; CORS only for allowed origins |
| AI | No sources → no call; invisible content never sent (assert on the stub's request); billed vs unbilled refunds |
| Cron | Call the job with a fixed `now`; rows attached meanwhile survive; R2 objects deleted with their rows |
| Errors | Unknown path → JSON 404; unexpected error → generic 500 without internals |

```ts
// Cron: test the function directly with a fixed clock.
it("deletes only unattached uploads older than 7 days", async () => {
  // …insert one old unattached file, one old attached file, one new unattached file
  const deleted = await deleteUnattachedFiles(env, 7, Date.parse("2026-01-10T00:00:00Z"))
  expect(deleted).toBe(1)
  expect(await env.FILES.get("files/old-unattached")).toBeNull()
})
```

## 3. Migration tests

A migration that changes existing data is tested twice: on a **fresh** database (all migrations) and on a **production-like** one (migrations up to the previous one, realistic rows inserted, then the new one applied).

```ts
import { applyD1Migrations, env as bindings } from "cloudflare:test"
import type { D1Migration } from "cloudflare:test"

const all = (bindings as unknown as { TEST_MIGRATIONS: D1Migration[] }).TEST_MIGRATIONS
const db = (bindings as unknown as Env).DB

it("0016 keeps current memberships as current stints", async () => {
  await applyD1Migrations(db, all.filter((m) => m.name < "0016"))
  await db.batch([
    "INSERT INTO users VALUES ('u1','a@x.test','A','admin','t','t')",
    "INSERT INTO projects VALUES ('p1','Apollo','t')",
    "INSERT INTO project_members VALUES ('p1','u1')",
  ].map((sql) => db.prepare(sql)))

  await applyD1Migrations(db, all) // applies only what's missing

  const rows = await db.prepare("SELECT project_id, user_id FROM project_members").all()
  expect(rows.results).toEqual([{ project_id: "p1", user_id: "u1" }])
})
```

Check row counts before and after for every table the migration touches. A migration test file runs alone (its own database), so it doesn't disturb the API tests.

## 4. Front-end checks

- `pnpm typecheck` and `pnpm lint` catch most front-end mistakes; keep them at zero warnings.
- Pure logic in `src/lib/` (parsers, formatters, route parsing) can get unit tests. Add a second Vitest project for `src/` only if the owner approves the setup (jsdom is a new dependency).
- Every screen is checked by hand or with Playwright for its **empty, loading, error and success** states. The States example page shows the shared ones.
- `pnpm build` must pass; it typechecks every project and bundles.

### React Doctor (optional static React check)

[react-doctor](https://github.com/millionco/react-doctor) (MIT, from millionco) scans React code without running it for state and effect misuse, performance, architecture, security, accessibility and maintainability. It complements `pnpm lint` and `code-reviewer`; it doesn't replace them, a code review, or the React DevTools Profiler.

- **Owner approval first**, like any dependency: `npx` downloads and runs a package. Once approved, pin a version instead of `@latest`.
- **Run it on what changed:** `npx react-doctor@latest --no-telemetry src/a.tsx src/b.tsx`, or with no file names for the whole project. `--format json` gives machine-readable output. Rules are configured in `doctor.config.ts` at the project root.
- **Telemetry:** by default it reports crashes, run traces and anonymous counters (versions, framework, project size, rule names and counts, no file contents) to Sentry. Always pass `--no-telemetry` here.
- **Findings are leads, not verdicts.** Where it disagrees with our rules (shadcn on Base UI, `DESIGN_SYSTEM.md`, `SECURITY.md`), ours win: configure or suppress the rule rather than bending the code.
- **`install` and `ci install` need a separate owner OK.** `install` adds an agent skill (Claude Code, Cursor, Codex, OpenCode) and `ci install` adds a pull request workflow that posts a summary comment. Its README doesn't list the files either one writes, so run them on a throwaway branch and read `git status` and the diff first. `devops` decides whether the workflow fits `docs/DEPLOY_AND_CI.md`.
- Checked on 30 October 2026 against its README and docs. They list React, Vite and Preact but don't state React 19, so treat the first run on this starter as a trial.

## 5. Browser QA with Playwright (text first)

Playwright is a devDependency. Install a browser once: `pnpm exec playwright install chromium` (or use an installed Chrome with `channel: "chrome"`). Keep QA scripts in a temporary folder, not in the repo.

**Text checks first.** Verify with DOM text, roles, computed styles, bounding boxes, overflow, focus and console errors. **Never PNG.** A screenshot only when text can't tell, as JPEG at quality 60 with CSS scale, and only a few per task.

```js
// qa.mjs: run with `node qa.mjs` while `pnpm dev` is running
import { chromium } from "playwright"

const BASE = "http://localhost:5173"
const SIZES = [[320, 640], [375, 760], [768, 1024], [1280, 800], [1920, 1080]]
const browser = await chromium.launch()

for (const colorScheme of ["light", "dark"]) {
  for (const [width, height] of SIZES) {
    const context = await browser.newContext({ viewport: { width, height }, colorScheme, reducedMotion: "reduce" })
    const page = await context.newPage()
    const errors = []
    page.on("console", (m) => m.type() === "error" && errors.push(m.text()))
    page.on("pageerror", (e) => errors.push(e.message))

    await page.goto(`${BASE}/`)
    await page.getByRole("heading", { level: 1 }).waitFor()

    const result = await page.evaluate(() => ({
      h1: [...document.querySelectorAll("h1")].map((h) => h.textContent?.trim()),
      overflow: document.documentElement.scrollWidth > document.documentElement.clientWidth,
      title: document.title,
      // Targets under 24 px (WCAG 2.5.8)
      smallTargets: [...document.querySelectorAll("a, button, [role=button], input, select, textarea")]
        .map((el) => ({ el, r: el.getBoundingClientRect() }))
        .filter(({ r }) => r.width > 0 && (r.width < 24 || r.height < 24))
        .map(({ el }) => el.getAttribute("aria-label") || el.textContent?.trim()),
      // Filled brand buttons per page (should be at most one per area)
      primaryButtons: document.querySelectorAll("[data-slot=button].bg-primary").length,
    }))
    console.log(colorScheme, width, JSON.stringify(result), errors.length ? `errors: ${errors}` : "no console errors")

    // Keyboard: the skip link is the first Tab stop, and focus is visible.
    await page.keyboard.press("Tab")
    console.log("first focus:", await page.evaluate(() => document.activeElement?.textContent?.trim()))

    // Only if text can't answer a visual question:
    // await page.screenshot({ path: `/tmp/qa-${colorScheme}-${width}.jpg`, type: "jpeg", quality: 60, scale: "css" })
    await context.close()
  }
}
await browser.close()
```

What `qa-tester` covers for a change: the happy path, then edge cases (empty, very long text, many items, slow network with `page.route` delays, invalid input, refresh mid-flow, Back); sizes 320, 375, 768, 1280, 1920; light and dark; console and failed requests; keyboard (Tab order, visible focus, Enter/Space/Esc, no traps, focus on the new page's `h1` after navigation); reduced motion; every role that sees the change; and the **UX/UI review checklist** in `DESIGN_SYSTEM.md` §23.

## 6. Accessibility (axe)

Zero axe violations is the bar, in both themes. `@axe-core/playwright` is a new dependency: ask the owner before adding it. With it:

```js
import AxeBuilder from "@axe-core/playwright"

const results = await new AxeBuilder({ page }).withTags(["wcag2a", "wcag2aa", "wcag21a", "wcag21aa", "wcag22aa"]).analyze()
for (const v of results.violations) console.log(v.id, v.impact, v.nodes.length, v.helpUrl)
```

Without it, inject `axe-core` from `node_modules` with `page.addScriptTag({ path })` and call `axe.run()`. Report violations by WCAG 2.2 criterion with the selector. Contrast is computed from the tokens in `src/index.css` for both themes, not from screenshots.

## 7. Before calling it done

- [ ] `pnpm typecheck`, `pnpm lint`, `pnpm test`, `pnpm build` pass (full runs once at the end; focused runs while working: `pnpm vitest run worker/test/items.test.ts`).
- [ ] New server behaviour has tests, including a refused caller and, for writes, a race.
- [ ] Migrations tested fresh and on existing data.
- [ ] UI checked at the five widths, light and dark, keyboard, reduced motion, no console errors, zero axe violations.
- [ ] `DESIGN_SYSTEM.md` §23 checklist walked (`ux-reviewer` for UI changes).
- [ ] If React Doctor is set up (§4), it was run on the changed `src/` files and every finding is fixed, configured away with a reason, or noted as a nit.
- [ ] No PNGs produced; at most a few JPEGs.
