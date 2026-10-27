# Browser extension (optional module)

A Manifest V3 extension for Chrome and Edge that talks to the app's Worker: it adds an "Add to {{PROJECT_NAME}}" action where people already work (a chat attachment, a Drive file, a link on any page) and uploads the file with a narrow personal token. Use this module only if the product needs it. The server side (tokens, CORS) is in [`SECURITY.md` §5](./SECURITY.md#5-scoped-bearer-tokens-and-cors).

## Contents
1. [Shape](#1-shape)
2. [Pinned key and extension ID](#2-pinned-key-and-extension-id)
3. [Connecting: the token hand-over](#3-connecting-the-token-hand-over)
4. [Narrow token scope](#4-narrow-token-scope)
5. [Dev vs store builds](#5-dev-vs-store-builds)
6. [Content-script buttons](#6-content-script-buttons)
7. [Permissions](#7-permissions)
8. [Testing](#8-testing)

## 1. Shape

```text
extension/
├── public/manifest.json     MV3 manifest (with the pinned "key")
├── public/icons/            16, 32, 48, 128 px
├── src/background.ts        Service worker: token storage, API calls, context menus, onMessageExternal
├── src/content/<site>.ts    One content script per site: adds buttons, asks the service worker to act
├── src/popup.tsx            Status panel: account, how to add, last 5 additions, settings link
├── src/options.tsx          Settings: server, Connect, paste a token, Disconnect
├── src/lib/api.ts           Typed calls to the Worker (keep in step with shared/api.ts)
├── build.mjs                Vite build of each entry; --dev adds localhost; zip refuses dev builds
└── test/                    A mock API server and a Playwright e2e that loads the built extension
```

- Reuse the app's theme (`src/index.css`) and a few shadcn components (`Button`, `Input`, `Label`, `Badge`) so the popup looks like the app; light and dark follow the system.
- Name the product and the action separately and keep both: "Save to {{PROJECT_NAME}}" (the extension) and "Add to {{PROJECT_NAME}}" (what the buttons do).
- Scripts in `package.json`: `ext:build`, `ext:build --dev`, `ext:zip`, `ext:typecheck`.

## 2. Pinned key and extension ID

Put the **public** half of an RSA 2048 key in `manifest.json` `"key"`, so every unpacked install has the same ID. The Worker only trusts known IDs (`EXTENSION_IDS` in `wrangler.jsonc`), and the app's connect page hands a token only to those.

```bash
# Once: make the key pair; keep key.pem in the team's password manager, never in git (*.pem is ignored).
openssl genrsa 2048 | openssl pkcs8 -topk8 -nocrypt -out key.pem
openssl rsa -in key.pem -pubout -outform DER | openssl base64 -A   # → manifest "key"
# The ID it produces: load unpacked and read it on chrome://extensions, or compute it from the DER.
```

- **Publishing to the Chrome Web Store or Edge Add-ons gives a new ID.** Add it to `EXTENSION_IDS` (comma-separated) before people install the store version.
- Changing the `key` changes the ID; update `EXTENSION_IDS` with it.
- Expose the allowed IDs to the app with a public `GET /api/extension/allowed`, so the connect page can refuse unknown extensions.

## 3. Connecting: the token hand-over

One click, no copy and paste, and only between known parties:

1. Settings page → **Connect**. The extension opens `<server>/extension/connect?extensionId=<its id>`.
2. The app's connect page (signed in with the normal session) checks `extensionId` against `/api/extension/allowed`, shows which extension is asking (name and ID), and on **Confirm** creates a token (`POST /api/extension/tokens`, a real session is required: a token can't create tokens).
3. It sends the token with `chrome.runtime.sendMessage(extensionId, { type: "app-token", token, name })`. Only origins listed in the manifest's `externally_connectable.matches` can do this.
4. The service worker checks the sender's origin, checks the token's shape, verifies it with `GET /api/extension/me`, and only then stores it. A token that would switch the server or replace a working connection to another account is kept aside until the person confirms in settings.
5. If the hand-over fails, the connect page deletes the token again.

```ts
// background.ts
chrome.storage.local.setAccessLevel({ accessLevel: "TRUSTED_CONTEXTS" }) // content scripts can't read the token

chrome.runtime.onMessageExternal.addListener((message, sender, sendResponse) => {
  const origin = sender.origin ?? (sender.url ? new URL(sender.url).origin : "")
  if (!CONNECT_ORIGINS.includes(origin) || !isTokenMessage(message)) {
    sendResponse({ ok: false, error: "Not accepted" })
    return
  }
  const token = message.token.trim()
  if (!/^ext_[A-Za-z0-9_-]{20,100}$/.test(token)) {
    sendResponse({ ok: false, error: "That doesn't look like a token." })
    return
  }
  void (async () => {
    try {
      const me = await getMe(origin, token) // a bad token must not replace a working one
      await saveConnection({ server: origin, token, name: me.name })
      sendResponse({ ok: true, name: me.name })
    } catch (error) {
      sendResponse({ ok: false, error: errorMessage(error) })
    }
  })()
  return true // keep the channel open for the async response
})
```

**Fallback:** "Or paste a token" in settings (and in the popup), for servers that `externally_connectable` can't list. The person creates the token in the app's settings.

Storage rules: the token lives only in `chrome.storage.local` (never `sync`), is never logged, and is sent only as `Authorization: Bearer …` to the chosen server. **Disconnect** removes it locally; revoking it in the app stops it everywhere. Any 401 switches the popup to **Reconnect**.

## 4. Narrow token scope

The server accepts the token on an explicit allow-list of routes only ([`SECURITY.md` §5](./SECURITY.md#5-scoped-bearer-tokens-and-cors)):

| Route | Purpose |
| --- | --- |
| `GET /api/extension/me` | Who am I, and may I add things (`canAdd`) |
| `POST /api/files` | Upload one file (multipart `file`, optional `originalUrl` https only, `originalName`) |

Everything else answers 401 to a token. Uploads with a token are rate-limited per person per hour (429 with `Retry-After`, exposed through CORS), and unattached uploads are deleted by the cron job, so a leaked token can do very little. After upload, the extension opens `<server>/<new-item-page>?file=<id>` in a new tab, where the person finishes (for example, an AI draft they review).

## 5. Dev vs store builds

```js
// build.mjs (excerpt)
const dev = process.argv.includes("--dev")
const DEV_ORIGIN = "http://localhost:5173/*"
if (dev && process.argv.includes("--zip")) throw new Error("Don't pack a --dev build: it accepts tokens from localhost.")

// Vite define: __DEV_BUILD__ lets the settings page list "Local development".
const define = { __DEV_BUILD__: JSON.stringify(dev) }

// The manifest for dev builds also accepts the hand-over from the local dev server.
if (dev) {
  const matches = manifest.externally_connectable.matches
  if (!matches.includes(DEV_ORIGIN)) matches.push(DEV_ORIGIN)
}
```

- A normal build only accepts tokens from the live app; `--dev` also from `http://localhost:5173`. Never ship a dev build.
- Version in `manifest.json` goes up with every store upload.
- Load unpacked: `chrome://extensions` (or `edge://extensions`) → Developer mode → Load unpacked → `extension/dist`. Reload the card after each rebuild.

## 6. Content-script buttons

- One script per site, matched narrowly (`https://chat.example.com/channels/*`), `run_at: "document_idle"`.
- Site markup isn't public and changes: find things by stable signals (URLs of attachments, element ids built from message ids, `aria-selected`, `data-id`) rather than generated class names, and fail quietly (no button) when they change.
- Buttons appear on hover **and on keyboard focus** (and always on touch screens); their accessible name is the visible text plus the file ("Add to {{PROJECT_NAME}}: handbook.pdf").
- States in words: **Adding…** → **Added ✓** (opens the app) or **Not added** with the reason underneath; announced with `role="status"`.
- Errors people can act on: over the size limit, a type the app doesn't take, a link that is really a web page (sign-in page), no access, an expired link ("reload the page"), 401 (Reconnect), 403 (your role can't add), 429 (with the wait), offline.
- Content scripts never hold the token; they ask the service worker (`chrome.runtime.sendMessage`) to download and upload.
- Right-click menu items ("Add file to …", "Add image to …") for any page, shown only to connected people who may add.

## 7. Permissions

Ask for as little as possible, and explain each one in the extension's README and store listing:

| Permission | Why |
| --- | --- |
| `storage` | Server address, token, the last five additions (local only) |
| `contextMenus` | "Add file / image to …" on links and images |
| Host access for the specific sites with buttons | The buttons, and downloading those sites' files with the person's own session |
| `optional_host_permissions: ["https://*/*"]` | Never granted up front; requested one site at a time (`chrome.permissions.request`) when someone uses the menu there |
| `externally_connectable` (not a permission) | Lets only the live app (plus localhost in dev builds) send the token |

No `activeTab`/`scripting` unless the extension really reads pages. No analytics. Privacy note: the extension reads nothing until the person chooses to add a file, and sends it only to their own server.

## 8. Testing

- `test/mock-server.mjs`: a stand-in API with a known test token and sample files (a PDF, a PNG, an oversized file, a "PDF" that is really HTML).
- `test/e2e.mjs`: Playwright loads the built extension in Chromium (`--disable-extensions-except`, `--load-extension`) and checks connecting (wrong and right tokens), the popup, buttons on fixture pages (shown on hover and focus, names, Enter adds the file), the menu, too big, wrong type, 429, 401 → Reconnect, offline, the connect page's hand-over and that other sites can't send tokens. Text checks first; JPEG only when needed.
- Server tests for the token routes and CORS live with the Worker tests ([`TESTING.md`](./TESTING.md)).
