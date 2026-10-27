# Backend on Cloudflare

How the API is built: one Cloudflare Worker with Hono serving `/api/*`, the React app as static assets, and Cloudflare's storage and AI products behind it. These patterns come from production apps on this stack (a knowledge app with accounts, files and Claude answers; a realtime chat on Durable Objects; a small app on Workers AI), generalised. Security rules are in [`SECURITY.md`](./SECURITY.md), AI in [`AI_FEATURES.md`](./AI_FEATURES.md), tests in [`TESTING.md`](./TESTING.md).

## Contents
1. [Worker and Hono structure](#1-worker-and-hono-structure)
2. [D1: schema, migrations, queries](#2-d1-schema-migrations-queries)
3. [R2: files](#3-r2-files)
4. [KV vs D1 vs Durable Objects](#4-kv-vs-d1-vs-durable-objects)
5. [Durable Objects and WebSockets](#5-durable-objects-and-websockets)
6. [Workers AI vs the Claude API](#6-workers-ai-vs-the-claude-api)
7. [Cron Triggers](#7-cron-triggers)
8. [Env vars vs secrets](#8-env-vars-vs-secrets)
9. [Local dev and remote](#9-local-dev-and-remote)
10. [wrangler.jsonc reference](#10-wranglerjsonc-reference)

## 1. Worker and Hono structure

```text
worker/
├── index.ts          The app: middleware order, routes, onError, the scheduled handler
├── http.ts           HttpError, fail(), tooMany(), readJson()/validate() with zod, requireRole(), now(), newId()
├── auth.ts           Sessions, the auth and sameOrigin middleware (see SECURITY.md)
├── password.ts       PBKDF2 hashing, token hashing (WebCrypto only)
├── cleanup.ts        Cron housekeeping
├── <domain>.ts       Shared logic per domain (search.ts, quotas, AI calls)
├── routes/<area>.ts  One Hono sub-app per area (auth, people, items, files…)
├── env.d.ts          Optional secrets that `wrangler types` can't see
└── test/             Vitest in the Workers runtime (TESTING.md)
shared/api.ts         Request/response types and limits shared with the React app
migrations/           Numbered D1 SQL files
```

**Middleware order matters.** Public routes first, then everything that needs a session:

```ts
const app = new Hono<AppEnv>().basePath("/api")

// User data must never be cached by the browser or anything in between.
app.use("*", async (c, next) => {
  await next()
  c.res.headers.set("Cache-Control", "private, no-store")
})

app.get("/health", (c) => c.json({ ok: true }))
app.use("*", sameOrigin)        // CSRF: changes must come from our own pages
app.route("/", publicAuth)      // login, signup, logout: no session needed
app.use("*", auth)              // everything below needs a session
app.route("/", people)
app.route("/", items)
app.route("/", files)

app.notFound((c) => c.json({ error: "There's no API endpoint here." }, 404))

app.onError((error, c) => {
  if (error instanceof HttpError) {
    if (error.retryAfter !== null) c.header("Retry-After", String(error.retryAfter))
    return c.json({ error: error.message }, error.status as 400)
  }
  // Log for the owner; never send internals (stack traces, SQL) to the client.
  console.error("Unhandled API error:", error instanceof Error ? error.message : "unknown")
  return c.json({ error: "Something went wrong on our side. Try again in a moment." }, 500)
})

// HTTP goes to Hono; the Cron Trigger runs housekeeping.
export default {
  fetch: app.fetch,
  async scheduled(_controller: ScheduledController, env: Env, ctx: ExecutionContext) {
    ctx.waitUntil(deleteUnattachedFiles(env).catch((e) => console.error("Cleanup failed:", e instanceof Error ? e.message : "unknown")))
  },
} satisfies ExportedHandler<Env>
```

**`worker/http.ts` helpers** (every handler uses them):

```ts
export type AppEnv = {
  Bindings: Env
  Variables: { user: CurrentUser; sessionHash: string | null; viaToken?: boolean }
}

/** An error the client should see, as `{ error }` with this status. */
export class HttpError extends Error {
  readonly status: number
  readonly retryAfter: number | null // seconds; sent as Retry-After
  constructor(status: number, message: string, retryAfter: number | null = null) {
    super(message)
    this.status = status
    this.retryAfter = retryAfter
  }
}

export function fail(status: number, message: string): never {
  throw new HttpError(status, message)
}

/** 429 with Retry-After: seconds until the window that started at `windowStart` frees up. */
export function tooMany(message: string, windowStart: string | null | undefined, windowMs: number): never {
  const start = windowStart ? Date.parse(windowStart) : NaN
  const ms = Number.isFinite(start) ? start + windowMs - Date.now() : windowMs
  throw new HttpError(429, message, Math.max(1, Math.ceil(ms / 1000)))
}

/** Parses and validates a JSON body. Bad input is a 400 naming the field. */
export async function readJson<T extends z.ZodType>(c: AppContext, schema: T): Promise<z.infer<T>> {
  let raw: unknown
  try {
    raw = await c.req.json()
  } catch {
    fail(400, "The request body must be valid JSON.")
  }
  const parsed = schema.safeParse(raw)
  if (!parsed.success) {
    const issue = parsed.error.issues[0]
    const field = issue?.path.join(".")
    fail(400, field ? `Check "${field}": ${issue.message}` : (issue?.message ?? "Check the request and try again."))
  }
  return parsed.data
}

export function requireRole(user: CurrentUser, ...roles: Role[]) {
  if (!roles.includes(user.role)) fail(403, "You don't have permission to do that.")
}

/** Escapes % and _ for LIKE ... ESCAPE '\'. */
export const likePattern = (q: string) => `%${q.replace(/[\\%_]/g, (m) => `\\${m}`)}%`

export const now = () => new Date().toISOString()
export const newId = () => crypto.randomUUID()
```

**A route file** validates input with zod, checks permission, does the work, returns the shared type:

```ts
export const items = new Hono<AppEnv>()

const renameSchema = z.object({ title: z.string().trim().min(1).max(LIMITS.title), revision: z.number().int() })

// PATCH /api/items/:id
items.patch("/items/:id", async (c) => {
  const user = c.get("user")
  const input = await readJson(c, renameSchema)
  const result = await c.env.DB.prepare(
    `UPDATE items SET title = ?1, revision = revision + 1, updated_at = ?2
     WHERE id = ?3 AND revision = ?4 AND (created_by = ?5 OR ?6 = 'admin')`,
  )
    .bind(input.title, now(), c.req.param("id"), input.revision, user.id, user.role)
    .run()
  if (result.meta.changes === 0) {
    // Tell "not yours / not there" (404) apart from "changed meanwhile" (409).
    const exists = await c.env.DB.prepare("SELECT revision FROM items WHERE id = ?").bind(c.req.param("id")).first()
    if (!exists) fail(404, "That item doesn't exist or you can't edit it.")
    fail(409, "Someone changed this item meanwhile. Reload to see their changes.")
  }
  return c.json<ItemResponse>(await loadItem(c.env.DB, c.req.param("id"), user))
})
```

Conventions:
- Error messages are full sentences for people ("What happened. What to do."), never codes.
- 404 rather than 403 when the caller may not even know the thing exists (no existence leaks).
- Limits (lengths, sizes, counts) live in `shared/api.ts` (`LIMITS`), used by zod on the server and by the form on the client.

## 2. D1: schema, migrations, queries

### Migrations
- One numbered file per change: `migrations/0001_init.sql`, `0002_<what>.sql`, … Apply locally with `pnpm db:migrate:local`, remotely with `pnpm db:migrate:remote` (owner's go-ahead, after a backup).
- Four digits, no gaps, next number only (`ls migrations | tail -1`). Wrangler records applied migrations **by file name** in `d1_migrations`, so **never rename or renumber a migration once it has been applied anywhere** (local counts too): it would run again. Two branches that both added `0007_…`: the one merged second renumbers its file *before* it's applied outside that branch.
- **Never edit a migration that has been applied anywhere.** Fix forward with a new one.
- Start each file with a comment saying why it exists and what it changes on existing data.
- Drop tables in their own migration once nothing references them, with a comment.
- Test migrations on a fresh database **and** on a production-like one with data from before the change ([`TESTING.md`](./TESTING.md)).

```sql
-- 0001_init.sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('admin', 'manager', 'user')),
  created_at TEXT NOT NULL,
  last_seen_at TEXT
);

CREATE TABLE items (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  revision INTEGER NOT NULL DEFAULT 1,      -- optimistic concurrency (409 on stale writes)
  created_by TEXT NOT NULL REFERENCES users (id),
  created_at TEXT NOT NULL,                 -- ISO strings: sortable, readable
  updated_at TEXT NOT NULL
);
CREATE INDEX items_created_by ON items (created_by, updated_at);
```

Times are ISO strings (`strftime('%Y-%m-%dT%H:%M:%fZ', 'now')` in SQL, `new Date().toISOString()` in TS). IDs are `crypto.randomUUID()`.

### Views with INSTEAD OF triggers: change a table without touching every query
When a table's meaning changes (memberships gaining a history, for example), keep the old name as a **view** so every reader stays correct, and route writes through triggers:

```sql
-- Each stint in a project is kept (ended_at NULL = current). Only current
-- stints grant access, so project_members becomes a view of them.
CREATE TABLE project_memberships (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL REFERENCES projects (id) ON DELETE CASCADE,
  user_id TEXT NOT NULL REFERENCES users (id) ON DELETE CASCADE,
  started_at TEXT NOT NULL,
  ended_at TEXT,
  CHECK (ended_at IS NULL OR ended_at >= started_at)
);
-- At most one current stint per person and project.
CREATE UNIQUE INDEX project_memberships_current ON project_memberships (project_id, user_id) WHERE ended_at IS NULL;

INSERT INTO project_memberships (id, project_id, user_id, started_at, ended_at)
SELECT lower(hex(randomblob(16))), pm.project_id, pm.user_id, p.created_at, NULL
FROM project_members pm JOIN projects p ON p.id = pm.project_id;
DROP TABLE project_members;

CREATE VIEW project_members AS
  SELECT project_id, user_id, started_at FROM project_memberships WHERE ended_at IS NULL;

CREATE TRIGGER project_members_insert INSTEAD OF INSERT ON project_members
BEGIN
  INSERT INTO project_memberships (id, project_id, user_id, started_at, ended_at)
  SELECT lower(hex(randomblob(16))), NEW.project_id, NEW.user_id,
         COALESCE(NEW.started_at, strftime('%Y-%m-%dT%H:%M:%fZ', 'now')), NULL
  WHERE NOT EXISTS (SELECT 1 FROM project_memberships
                    WHERE project_id = NEW.project_id AND user_id = NEW.user_id AND ended_at IS NULL);
END;

CREATE TRIGGER project_members_delete INSTEAD OF DELETE ON project_members
BEGIN
  UPDATE project_memberships SET ended_at = MAX(started_at, strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
  WHERE project_id = OLD.project_id AND user_id = OLD.user_id AND ended_at IS NULL;
END;
```

A migration like this rewrites data: back up the remote database first.

### Full-text search with FTS5
```sql
-- Search index over published content only; rebuilt for an item when it's published or archived.
CREATE VIRTUAL TABLE chunks USING fts5 (
  title, heading, text,
  item_id UNINDEXED, version_id UNINDEXED, ordinal UNINDEXED,
  tokenize = 'porter unicode61'
);
```

**Never pass raw user text to `MATCH`** (FTS5 syntax errors and operator injection). Build the query: lower-case words, drop stopwords, quote each term, OR them together:

```ts
export function ftsQuery(text: string): string | null {
  const words = (text.toLowerCase().match(/[\p{L}\p{N}]+/gu) ?? []).filter((w) => w.length >= 2 && !STOPWORDS.has(w))
  const terms = [...new Set(words)].slice(0, 16)
  return terms.length ? terms.map((t) => `"${t}"`).join(" OR ") : null
}

const { results } = await db.prepare(
  `SELECT item_id, title, heading, snippet(chunks, 2, '', '', '…', 24) AS text
   FROM chunks WHERE chunks MATCH ?1 AND ${visibility.sql}
   ORDER BY bm25(chunks, 4.0, 2.0, 1.0)   -- title weighted highest
   LIMIT 20`,
).bind(match, ...visibility.params).all()
```

Apply visibility **inside** the query (the `WHERE` clause), never by filtering results afterwards: a `LIMIT` would otherwise hide allowed rows, and counts would leak.

### Batching and atomic guards
D1 has no interactive transactions. Two tools replace them:

- **`db.batch([...])`** runs statements in one implicit transaction: all or nothing.
- **Guards inside a single statement**, so concurrent requests can't both pass a check:

```ts
// Limit check and insert in ONE statement: parallel requests can't pass the limit together.
const result = await db.prepare(
  `INSERT INTO tokens (id, user_id, token_hash, label, created_at)
   SELECT ?1, ?2, ?3, ?4, ?5
   WHERE (SELECT COUNT(*) FROM tokens WHERE user_id = ?2) < ?6`,
).bind(id, user.id, hash, label, now(), MAX_TOKENS).run()
if (result.meta.changes === 0) fail(409, `You already have ${MAX_TOKENS} tokens. Revoke one you don't use, then try again.`)

// Decide the role inside the INSERT, so two first sign-ups can't both become admin.
const NEW_USER_ROLE_SQL = "CASE WHEN EXISTS (SELECT 1 FROM users WHERE role = 'admin') THEN 'user' ELSE 'admin' END"

// Delete-and-return with the condition re-checked, so a row that changed meanwhile is kept.
const { results } = await db.prepare(
  `DELETE FROM files WHERE id IN (SELECT id FROM files WHERE created_at < ?1 AND attached = 0 LIMIT ?2)
     AND attached = 0
   RETURNING r2_key`,
).bind(cutoff, 50).all<{ r2_key: string }>()
```

Check `result.meta.changes` after every guarded write. Use `RETURNING` to read what changed in the same statement. Fixed-window quotas use the same upsert shape ([`SECURITY.md`](./SECURITY.md#3-rate-limits-with-retry-after)).

### Backups before remote migrations
```bash
# Export the remote database (schema + data) before any migration that rewrites or drops data.
npx wrangler d1 export {{PROJECT_SLUG}} --remote --output backups/$(date +%Y%m%d-%H%M)-before-00NN.sql
# D1 Time Travel can also restore to a point in the last 30 days:
npx wrangler d1 time-travel info {{PROJECT_SLUG}}
npx wrangler d1 time-travel restore {{PROJECT_SLUG}} --timestamp=<ISO time before the migration>
```

Keep `backups/` out of git (`.gitignore`); it holds personal data.

### Seeds and demo accounts
Seeds fill local and demo databases; they are **never** run against production. Content comes from real public sources that fit the product (note the source in a comment), never invented filler and never real personal data ([`WORKFLOW.md` §9](./WORKFLOW.md#9-seed-and-demo-data)).

Every seed is **idempotent**: fixed ids plus `INSERT OR IGNORE` (keep what's there) or `ON CONFLICT … DO UPDATE` (refresh it), so running it twice changes nothing and it can be re-run after a new migration.

```sql
-- scripts/seed-local.sql: local and demo only. Run: npx wrangler d1 execute DB --local --file scripts/seed-local.sql
-- Source: <public site or document the content comes from>, checked <date>.
INSERT OR IGNORE INTO users (id, email, name, role, created_at) VALUES
  ('demo-admin',   'admin@demo.test',   'Demo admin',   'admin',   '2026-01-01T00:00:00.000Z'),
  ('demo-manager', 'manager@demo.test', 'Demo manager', 'manager', '2026-01-01T00:00:00.000Z'),
  ('demo-user',    'user@demo.test',    'Demo user',    'user',    '2026-01-01T00:00:00.000Z');

INSERT INTO items (id, title, created_by, created_at, updated_at) VALUES
  ('seed-item-1', '<a real public title>', 'demo-manager', '2026-01-01T00:00:00.000Z', '2026-01-01T00:00:00.000Z')
ON CONFLICT (id) DO UPDATE SET title = excluded.title, updated_at = excluded.updated_at;
```

**Demo accounts, one per role.** When `DEMO_MODE` is on (local and the demo Worker only), the sign-in page shows one button per role that signs in as the matching seeded account. The server enforces it, not the UI:

```ts
// worker/routes/auth.ts
app.post("/api/auth/demo", async (c) => {
  if (!demoMode(c.env)) fail(404, "Page not found.")  // invisible in production
  const { role } = await readJson(c, z.object({ role: z.enum(["admin", "manager", "user"]) }))
  const user = await c.env.DB.prepare("SELECT * FROM users WHERE id = ?1").bind(`demo-${role}`).first()
  if (!user) fail(404, "Demo data isn't loaded. Run the seed, then try again.")
  return startSession(c, user) // your normal sign-in path: new session row + cookie (SECURITY.md §1)
})
```

The client learns whether to show the buttons from a public config endpoint (e.g. `GET /api/config` → `{ demoMode: true }`), never from a `VITE_*` flag baked into the build. A test asserts the route answers 404 when `DEMO_MODE` is `false`.

## 3. R2: files

Bytes go in R2, metadata in D1 (`files`: id, r2_key, name, mime, size, uploaded_by, created_at). Never store file bytes in D1.

### Streaming upload with a hard size cap
Never read more than the limit: require `Content-Length`, reject early, then **count bytes as they stream** in case the header lies.

```ts
const MULTIPART_OVERHEAD = 64 * 1024
const TOO_BIG = "That file is too big. The limit is 10 MB."

files.post("/files", async (c) => {
  const user = c.get("user")
  requireRole(user, "admin", "manager")
  const maxBody = LIMITS.fileBytes + MULTIPART_OVERHEAD
  const lengthHeader = c.req.header("Content-Length")
  if (!lengthHeader) fail(411, "The upload needs a Content-Length header.")
  const length = Number(lengthHeader)
  if (!Number.isFinite(length) || length < 0) fail(400, "The Content-Length header isn't valid.")
  if (length > maxBody) fail(413, TOO_BIG)
  const contentType = c.req.header("Content-Type") ?? ""
  if (!c.req.raw.body || !contentType.toLowerCase().startsWith("multipart/form-data")) {
    fail(400, 'Send the file as multipart form data in a field called "file".')
  }
  let received = 0
  const limited = c.req.raw.body.pipeThrough(
    new TransformStream<Uint8Array, Uint8Array>({
      transform(chunk, controller) {
        received += chunk.byteLength
        if (received > maxBody) controller.error(new HttpError(413, TOO_BIG))
        else controller.enqueue(chunk)
      },
    }),
  )
  let form: FormData
  try {
    form = await new Response(limited, { headers: { "Content-Type": contentType } }).formData()
  } catch {
    if (received > maxBody) fail(413, TOO_BIG)
    fail(400, 'Send the file as multipart form data in a field called "file".')
  }
  const file = form.get("file")
  if (!file || typeof file === "string") fail(400, 'Send the file as multipart form data in a field called "file".')
  if (file.size === 0) fail(400, "That file is empty.")

  const name = sanitizeName(file.name)
  const ext = name.includes(".") ? name.split(".").pop()!.toLowerCase() : ""
  const mime = FILE_TYPES[ext] // allow-list in shared/api.ts: { pdf: "application/pdf", png: "image/png", … }
  if (!mime) fail(400, "That file type isn't supported. Use PNG, JPG, WEBP, PDF, DOCX, PPTX, TXT or MD.")
  const bytes = new Uint8Array(await file.arrayBuffer())
  if (!matchesType(ext, bytes.subarray(0, 8192))) fail(400, `That file's contents don't look like a .${ext} file.`)

  const id = newId()
  const key = `files/${id}`
  await c.env.FILES.put(key, bytes, { httpMetadata: { contentType: mime } })
  try {
    await c.env.DB.prepare("INSERT INTO files (id, r2_key, name, mime, size, uploaded_by, created_at) VALUES (?, ?, ?, ?, ?, ?, ?)")
      .bind(id, key, name, mime, bytes.byteLength, user.id, now()).run()
  } catch (error) {
    await c.env.FILES.delete(key) // no orphan objects
    throw error
  }
  return c.json<FileInfo>({ id, name, mime, size: bytes.byteLength }, 201)
})
```

### Type sniffing (magic bytes) and safe names
The extension and the browser's MIME type are claims; check the first bytes:

```ts
const startsWith = (b: Uint8Array, sig: number[], at = 0) => sig.every((x, i) => b[at + i] === x)
const ascii = (s: string) => [...s].map((ch) => ch.charCodeAt(0))

export function matchesType(ext: string, head: Uint8Array): boolean {
  switch (ext) {
    case "png": return startsWith(head, [0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a])
    case "jpg": case "jpeg": return startsWith(head, [0xff, 0xd8, 0xff])
    case "webp": return startsWith(head, ascii("RIFF")) && startsWith(head, ascii("WEBP"), 8)
    case "pdf": return startsWith(head, ascii("%PDF-"))
    case "docx": case "pptx": case "xlsx": return startsWith(head, [0x50, 0x4b, 0x03, 0x04]) // ZIP
    case "txt": case "md": return !head.includes(0) // text: no NUL bytes
    default: return false
  }
}

/** A readable file name without paths, control or reserved characters, at most 120 chars. */
export function sanitizeName(raw: string): string {
  const base = raw.split(/[\\/]/).pop() ?? ""
  // eslint-disable-next-line no-control-regex
  let name = base.replace(/[\u0000-\u001f\u007f<>:"|?*]/g, "").replace(/\s+/g, " ").trim().replace(/^\.+/, "")
  if (name.length > 120) {
    const dot = name.lastIndexOf(".")
    const ext = dot > 0 ? name.slice(dot) : ""
    name = name.slice(0, 120 - ext.length) + ext
  }
  return name || "file"
}
```

### Safe downloads
Check permission on **every** download (404 if not allowed), stream from R2, force download for anything that isn't a safe image, and sandbox it:

```ts
const encodeRfc5987 = (v: string) => encodeURIComponent(v).replace(/['()*]/g, (ch) => `%${ch.charCodeAt(0).toString(16).toUpperCase()}`)

files.get("/files/:id", async (c) => {
  const file = await loadFile(c.env.DB, c.req.param("id"))
  if (!file || !(await mayRead(c.env.DB, c.get("user"), file))) fail(404, "That file doesn't exist or you can't open it.")
  const object = await c.env.FILES.get(file.r2_key)
  if (!object) fail(404, "That file is missing from storage.")
  const inline = IMAGE_EXTS.has(extOf(file.name)) && c.req.query("download") !== "1"
  const fallback = file.name.replace(/[^\x20-\x7e]|["\\]/g, "_")
  return new Response(object.body, {
    headers: {
      "Content-Type": file.mime,
      "Content-Length": String(object.size),
      "Content-Disposition": `${inline ? "inline" : "attachment"}; filename="${fallback}"; filename*=UTF-8''${encodeRfc5987(file.name)}`,
      "X-Content-Type-Options": "nosniff",
      "Content-Security-Policy": "default-src 'none'; sandbox",
    },
  })
})
```

### Cleanup cron
Uploads that never got attached are deleted after N days by the daily Cron Trigger ([§7](#7-cron-triggers)): the D1 row goes first in one statement that re-checks "still unattached", then the R2 objects (`env.FILES.delete(keys)` takes up to 1,000 keys). Batches of 50, at most 20 batches a run.

## 4. KV vs D1 vs Durable Objects

| Need | Use | Why |
| --- | --- | --- |
| Relational data, permissions, search, anything you query or join | **D1** | SQL, indexes, FTS5, batches; strongly consistent per database |
| Files, images, exports, anything over ~100 KB | **R2** | Cheap blob storage; stream in and out |
| Read-heavy config, feature flags, cached lookups that can be ~60 s stale | **KV** | Global, fast reads; **eventually consistent**, last write wins. Never for counters, quotas, sessions you revoke, or anything read-after-write |
| One ordered authority per thing (a room, a document, a game), realtime fan-out, WebSockets, per-entity rate limits | **Durable Objects** | Single-threaded per id, strongly consistent storage (SQLite-backed), WebSocket hibernation |
| Cache of a deterministic, expensive response (AI output for the same input) | **Cache API** (`caches.default`) | Per data center, free, keyed by a URL you build |
| Background work, retries, fan-out jobs | **Queues** | At-least-once delivery, batching, retries |

Rules of thumb: start with D1. Add a Durable Object only for ordering or realtime. Use KV only for data that tolerates staleness. Quotas and rate limits belong in D1 (atomic upserts) or a Durable Object, not KV.

## 5. Durable Objects and WebSockets

The realtime pattern: **one Durable Object per room** (workspace, document), addressed by name, holding that room's state and its sockets. D1 stays the index (who is in which room); the DO is the ordered authority for the room's content.

```jsonc
// wrangler.jsonc
"durable_objects": { "bindings": [{ "name": "ROOMS", "class_name": "Room" }] },
"migrations": [{ "tag": "v1", "new_sqlite_classes": ["Room"] }]
```

**The Worker authenticates, checks membership in D1, then forwards** with the user id it stamped. The DO is reachable only through the Worker, so it can trust that header:

```ts
const roomFor = (env: Env, id: string) => env.ROOMS.get(env.ROOMS.idFromName(id))

app.get("/api/rooms/:id/socket", async (c) => {
  const user = c.get("user")
  const id = c.req.param("id")
  if (!(await isMember(c.env.DB, id, user.id))) return c.json({ error: "You're not in this room." }, 403)
  const headers = new Headers(c.req.raw.headers)
  headers.set("x-user-id", user.id) // stamped by the Worker, never taken from the client
  return roomFor(c.env, id).fetch(new Request("https://room/socket", { headers }))
})
```

**The Durable Object** uses the hibernation API (`ctx.acceptWebSocket`), keeps per-socket data in an attachment, loads state once, and validates every message:

```ts
import { DurableObject } from "cloudflare:workers"

type Attachment = { userId: string; status: "active" | "away" }

export class Room extends DurableObject<Env> {
  private state: RoomState = EMPTY

  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env)
    ctx.blockConcurrencyWhile(() => this.load()) // nothing runs before state is loaded
  }

  async fetch(request: Request) {
    const { pathname } = new URL(request.url)
    if (pathname === "/socket") {
      const userId = request.headers.get("x-user-id")
      if (!userId || request.headers.get("Upgrade") !== "websocket") return new Response("Expected a websocket", { status: 400 })
      const { 0: client, 1: server } = new WebSocketPair()
      this.ctx.acceptWebSocket(server) // hibernates when idle; no billing while asleep
      server.serializeAttachment({ userId, status: "active" } satisfies Attachment)
      return new Response(null, { status: 101, webSocket: client })
    }
    if (pathname === "/snapshot") return Response.json(this.visibleTo(request.headers.get("x-user-id")))
    return new Response("Not found", { status: 404 })
  }

  async webSocketMessage(socket: WebSocket, data: string | ArrayBuffer) {
    let message: ClientMessage
    try {
      message = JSON.parse(typeof data === "string" ? data : "") as ClientMessage
    } catch {
      return
    }
    const { userId } = socket.deserializeAttachment() as Attachment
    // Server-stamped identity: overwrite actor, author and time on every op; check ownership and limits.
    const op = stamp(message.op, userId, Date.now())
    const problem = checkOp(this.state, op)
    if (problem) return socket.send(JSON.stringify({ type: "reject", opId: op.id, error: problem }))
    const before = this.state
    this.state = applyOp(before, op)
    await this.save(before, this.state) // storage.put/delete take at most 128 keys per call
    this.broadcast({ type: "op", op })
  }

  async webSocketClose(socket: WebSocket, code: number) {
    this.broadcastPresence(socket)
    socket.close(code === 1005 ? 1000 : code, "Closing")
  }

  private broadcast(message: ServerMessage) {
    const text = JSON.stringify(message)
    for (const socket of this.ctx.getWebSockets()) socket.send(text)
  }
}
```

Client side: one socket per room; on close, reconnect with exponential backoff (0.5 s, 1 s, 2 s … up to 15 s) and show a "Reconnecting…" banner after ~1.5 s; after reconnecting, fetch `/snapshot` and replace local state, since ops may have been missed. Use the same reducer (`applyOp`) on the client for optimistic UI: apply locally, then reconcile with the echo. Ephemeral signals (typing, presence) are forwarded, never stored. Export the class from `worker/index.ts` (`export { Room } from "./room"`).

## 6. Workers AI vs the Claude API

| | Workers AI (`env.AI.run`) | Claude API (`@anthropic-ai/sdk`) |
| --- | --- | --- |
| Good for | Small, cheap, latency-sensitive transforms: transliteration, classification, short rewrites, embeddings | Answers people read and trust: grounded answers with citations, drafting from documents, structured extraction, reasoning |
| Setup | `"ai": { "binding": "AI" }`; no key | `ANTHROPIC_API_KEY` secret; model id in a var |
| Quality | Open models (e.g. Llama 3.3 70B); verify output | Highest quality; native citations, structured output, adaptive thinking |
| Cost control | Per-call neurons; cache identical inputs | Per-token; quotas per person, spend limit in the console |

Choose by quality first ([`AI_FEATURES.md`](./AI_FEATURES.md)): if people act on the output, use Claude.

**Workers AI pattern** (same-site only, strict input, cached by input, output verified):

```ts
const MODEL = "@cf/meta/llama-3.3-70b-instruct-fp8-fast"
const PROMPT_VERSION = 2 // bump when the prompt changes so old cached answers are ignored

async function suggest(request: Request, env: Env, ctx: ExecutionContext) {
  if (!sameSite(request)) return json({ error: "Forbidden" }, 403) // other sites can't spend our AI quota
  const text = String(((await request.json()) as { text?: unknown }).text ?? "").trim()
  if (!text || text.length > 80 || /[<>{}]/.test(text)) return json({ error: "Check the text and try again." }, 400)

  const cacheKey = new Request(`https://cache.internal/suggest?v=${PROMPT_VERSION}&q=${encodeURIComponent(text.toLowerCase())}`)
  const cached = await caches.default.match(cacheKey)
  if (cached) return cached

  const result = (await env.AI.run(MODEL, {
    messages: [{ role: "system", content: SYSTEM_PROMPT }, { role: "user", content: text }],
    temperature: 0,
    max_tokens: 120,
  })) as { response?: string }
  const output = (result.response ?? "").trim().split("\n")[0].trim()
  if (!looksRight(output)) return json({ error: "No suggestion." }, 422) // accept only output of the expected shape

  const response = json({ text: output }, 200, { "Cache-Control": "public, max-age=2592000" })
  ctx.waitUntil(caches.default.put(cacheKey, response.clone()))
  return response
}
```

**Claude client pattern** (a stub for tests, off without a key):

```ts
import Anthropic from "@anthropic-ai/sdk"

let testClient: ClaudeClient | null = null
/** Tests only: answer Claude calls with this stub (null to reset). */
export function setTestClaudeClient(client: ClaudeClient | null) { testClient = client }

/** A client, or null when no key is configured (AI is off; the feature degrades). */
export function claudeClient(env: Env, timeout: number): ClaudeClient | null {
  if (env.ENVIRONMENT === "test" && testClient) return testClient
  if (!env.ANTHROPIC_API_KEY) return null
  return new Anthropic({ apiKey: env.ANTHROPIC_API_KEY, maxRetries: 1, timeout })
}
```

Before writing any Claude code, read the `claude-api` skill for current model ids, parameters and pricing.

## 7. Cron Triggers

```jsonc
// Daily at 03:17 UTC (an odd minute avoids the top-of-hour rush).
"triggers": { "crons": ["17 3 * * *"] }
```

The `scheduled` handler (in [§1](#1-worker-and-hono-structure)) wraps work in `ctx.waitUntil`, logs a one-line result, and never throws unhandled. Jobs are **idempotent**, **batched** (small batches, a cap per run, the rest next time) and **safe under races** (re-check the condition in the delete). Test the job directly by calling the function with a fixed `now` ([`TESTING.md`](./TESTING.md)). Locally: `npx wrangler dev --test-scheduled` then `curl "http://localhost:8787/__scheduled?cron=17+3+*+*+*"`.

Typical jobs: delete unattached uploads, expire sessions and invites, clear old rate-limit windows, send digest emails, recompute caches.

## 8. Env vars vs secrets

| Kind | Where | Examples |
| --- | --- | --- |
| **Vars** (not secret, per environment) | `wrangler.jsonc` `vars`; locally overridden by `.dev.vars` | `ENVIRONMENT`, `ANTHROPIC_MODEL`, `ALLOWED_EMAIL_DOMAINS`, `SIGNUP_ENABLED`, `DEMO_MODE`, `EXTENSION_IDS` |
| **Secrets** | `npx wrangler secret put NAME` (production); `.dev.vars` (local, gitignored) | `ANTHROPIC_API_KEY`, `RESEND_API_KEY`, signing keys |
| **Client config** | Never secret: anything in the Vite bundle (including `VITE_*`) is public | Public URLs, feature labels |

- Type optional secrets in `worker/env.d.ts` (`wrangler types` only sees `vars`).
- Read boolean flags tolerantly and default to the safe value:

```ts
const envFlag = (value: unknown, fallback: boolean) => {
  const v = String(value ?? "").trim().toLowerCase()
  return v === "" ? fallback : v === "true" || v === "1" || v === "yes"
}
export const signupEnabled = (env: Env) => envFlag(env.SIGNUP_ENABLED, false) // off unless set
export const demoMode = (env: Env) => envFlag(env.DEMO_MODE, false)
```

- Production `vars` keep every demo or test switch off. A demo deployment is a **separate Worker** (`env.demo`) with its own database.
- Every name, its value per environment and how to set it: [`ENV_AND_SECRETS.md`](./ENV_AND_SECRETS.md).

## 9. Local dev and remote

| | Local (`pnpm dev`) | Remote (deployed) |
| --- | --- | --- |
| Worker | Runs inside Vite via `@cloudflare/vite-plugin` (one port: app + API) | `pnpm run deploy` |
| D1 | Local SQLite in `.wrangler/state` (`--local`) | The real database (`--remote`), owner's go-ahead |
| R2, KV, DO | Local simulations in `.wrangler/state` | Real resources |
| Vars and secrets | `.dev.vars` (copy of `.dev.vars.example`) | `vars` in `wrangler.jsonc` + `wrangler secret put` |
| Cron | `wrangler dev --test-scheduled` | Runs on schedule |

- Commands that touch data say which side: `wrangler d1 execute DB --local --command "…"` vs `--remote`. Default to `--local`; `--remote` needs the owner.
- Reset local data only with the owner's OK: `rm -rf .wrangler/state` then `pnpm db:migrate:local` (and any seed).
- Seed files for local demo data live in `scripts/seed-local.sql` (idempotent, [§2](#seeds-and-demo-accounts)); never run them against production.
- After any `wrangler.jsonc` change: `pnpm cf-typegen`.

## 10. wrangler.jsonc reference

Every binding type, commented. Uncomment only what the app uses; after any change run `pnpm cf-typegen`. Bindings are **not inherited** by `env.<name>` blocks: repeat them there.

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "{{PROJECT_SLUG}}",
  "main": "worker/index.ts",
  "compatibility_date": "2026-10-01",          // pin; bump deliberately and retest
  "compatibility_flags": ["nodejs_compat"],    // node:crypto, Buffer etc. for SDKs
  "workers_dev": true,                         // https://<name>.<subdomain>.workers.dev
  "preview_urls": false,                       // one address only
  // "routes": [{ "pattern": "app.example.com", "custom_domain": true }],

  // Static assets: the Vite build. /api/* reaches the Worker first; everything else is the SPA.
  "assets": {
    "not_found_handling": "single-page-application",
    "run_worker_first": ["/api/*"]
    // "binding": "ASSETS"                     // only if the Worker serves assets itself (env.ASSETS.fetch)
  },

  // D1 (SQLite). `npx wrangler d1 create {{PROJECT_SLUG}}`, paste the id.
  "d1_databases": [
    { "binding": "DB", "database_name": "{{PROJECT_SLUG}}", "database_id": "{{D1_DATABASE_ID}}", "migrations_dir": "migrations" }
  ],

  // R2 (files). `npx wrangler r2 bucket create {{PROJECT_SLUG}}-files`
  "r2_buckets": [{ "binding": "FILES", "bucket_name": "{{PROJECT_SLUG}}-files" }],

  // KV (eventually consistent key-value). `npx wrangler kv namespace create CACHE`
  // "kv_namespaces": [{ "binding": "CACHE", "id": "<id>" }],

  // Durable Objects (ordered state, WebSockets). Export the class from worker/index.ts.
  // "durable_objects": { "bindings": [{ "name": "ROOMS", "class_name": "Room" }] },
  // "migrations": [{ "tag": "v1", "new_sqlite_classes": ["Room"] }],   // DO class migrations (not D1)

  // Workers AI (no key needed).
  // "ai": { "binding": "AI" },

  // Vectorize (embeddings index). `npx wrangler vectorize create <name> --dimensions=768 --metric=cosine`
  // "vectorize": [{ "binding": "VECTORS", "index_name": "{{PROJECT_SLUG}}-index" }],

  // Queues (background jobs with retries).
  // "queues": {
  //   "producers": [{ "binding": "JOBS", "queue": "{{PROJECT_SLUG}}-jobs" }],
  //   "consumers": [{ "queue": "{{PROJECT_SLUG}}-jobs", "max_batch_size": 10, "max_retries": 3 }]
  // },

  // Rate limiting binding (per-key, per-location, approximate; D1 quotas stay the source of truth).
  // "ratelimits": [{ "name": "LIMITER", "namespace_id": "1001", "simple": { "limit": 100, "period": 60 } }],

  // Another Worker (service binding, no public hop).
  // "services": [{ "binding": "MAILER", "service": "{{PROJECT_SLUG}}-mailer" }],

  // Analytics Engine (high-volume event counts).
  // "analytics_engine_datasets": [{ "binding": "EVENTS", "dataset": "{{PROJECT_SLUG}}_events" }],

  // Browser Rendering (headless Chrome: PDFs, screenshots).
  // "browser": { "binding": "BROWSER" },

  // Hyperdrive (an external Postgres/MySQL).
  // "hyperdrive": [{ "binding": "PG", "id": "<id>" }],

  // Email sending (Cloudflare Email Service).
  // "send_email": [{ "name": "EMAIL" }],

  // Plain settings (not secret). Secrets: `npx wrangler secret put NAME`, locally .dev.vars.
  "vars": {
    "ENVIRONMENT": "production",
    "SIGNUP_ENABLED": "false",                 // invite-only in production
    "DEMO_MODE": "false"                       // never "true" in production
    // "ANTHROPIC_MODEL": "<model id from the claude-api skill>",
    // "ALLOWED_EMAIL_DOMAINS": "example.com",
    // "EXTENSION_IDS": "<32-letter extension id>"
  },

  // Cron Triggers (UTC).
  // "triggers": { "crons": ["17 3 * * *"] },

  // "limits": { "cpu_ms": 30000 },            // raise for heavy requests (paid plan)
  // "placement": { "mode": "smart" },         // run near the database when most time is spent there

  "observability": { "enabled": true },        // Workers Logs in the dashboard; `wrangler tail` live

  // A separate demo deployment: its own name, database and vars.
  // "env": {
  //   "demo": {
  //     "name": "{{PROJECT_SLUG}}-demo",
  //     "d1_databases": [{ "binding": "DB", "database_name": "{{PROJECT_SLUG}}-demo", "database_id": "<id>", "migrations_dir": "migrations" }],
  //     "vars": { "ENVIRONMENT": "demo", "DEMO_MODE": "true", "SIGNUP_ENABLED": "false" }
  //   }
  // }
}
```
