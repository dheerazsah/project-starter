# SEO and discoverability launch guide

How to make a public website show up on Google, Bing and AI answer engines (ChatGPT, Claude, Perplexity, Google AI Overviews). It is one self-contained file: copy it into any project as it is.

**When to use it:** at the end of production, when a public-facing site is about to launch (the same moment as `PRE_LAUNCH_COMPLIANCE_CHECKLIST.md`), and again after big content or URL changes. Internal tools, invite-only apps and admin areas should stay **out** of search: for those, only do step 0's "keep private pages out" items.

> **Agents:** don't act on this file until the owner says a public site is about to launch and asks for the SEO check. The one exception is the two early decisions below, which the `project-manager` records in the guide when the project starts.

## Why this file exists

> "I created my website using AI, but it's not showing up on Google."
> Submitted the site to Google Search Console? **No.** Submitted an XML sitemap? **No.** Optimised for SEO and GEO? **No.** Meta titles and descriptions? **No.** A Google Business Profile? **What's that?**

Building the site is half the job. Search engines must be told it exists, be able to read it, and understand what each page is about. Use the five questions as a quick self-test; the steps below answer each one.

| Question | Step |
| --- | --- |
| Can search engines read the site at all? | 0 |
| Submitted to Google Search Console (and Bing)? | 1 |
| XML sitemap submitted? | 2 |
| Meta titles and descriptions on every page? | 3 |
| Optimised for SEO and GEO? | 4 |
| Google Business Profile, if eligible? | 5 |

This is a practical checklist, not a ranking guarantee. Nobody can promise a position on Google, and anyone who does is selling something.

## Project details (fill in per project)

| Field | Value |
| ----- | ----- |
| Production domain (canonical, with or without `www`) | |
| Public pages that should be found (URLs) | |
| Private areas that must stay out (app, admin, demo, staging) | |
| Main search phrases people would type | |
| Business type: online-only, storefront, or visits customers | |
| Languages and countries | |
| Who owns Search Console and Bing Webmaster Tools access | |
| Review date and reviewer | |

## Two early decisions (at project start, not at launch)

These are the only parts to settle before building, because changing them late means rework:

1. **Does this product need to be found by search at all?** If no (an internal tool), the whole site stays `noindex` and you can skip the rest of this file except step 0.
2. **If yes, how do public pages reach crawlers as real HTML?** Google runs JavaScript, but slowly and not always; most AI crawlers don't run it at all and see an empty page. Public pages need their text and meta tags in the HTML the server sends: static or prerendered pages, server-side rendering, or (in the project starter) the Worker approach in the appendix.

---

## 0. Make sure search engines can read the site

AI-built sites most often fail here, before any SEO work matters.

- [ ] **Not blocked by accident:** no leftover `noindex` meta tag or `X-Robots-Tag` header on public pages; `robots.txt` doesn't `Disallow: /` in production.
- [ ] **Readable without JavaScript:** open a public page with JavaScript off (or `curl` it): the title, description, headings and main text are in the HTML.
- [ ] **Real 404s:** an unknown URL returns status 404 (or is `noindex`), not a 200 with the home page. Single-page apps often get this wrong, and Google reports it as "soft 404".
- [ ] **One address per page:** pick `https://` and either `www` or not; the others redirect (301) to it. Every page has a `<link rel="canonical">` pointing at its own preferred URL.
- [ ] **Hosting and firewall settings let crawlers in:** check your CDN or bot settings. On Cloudflare, check the AI bot policy (Search, Agent and Training crawlers can be allowed or blocked separately): blocking "Training" can also block Googlebot, Bingbot and Applebot because they're multi-purpose crawlers.
- [ ] **Keep private pages out:** signed-in app screens, admin, demo, staging and preview URLs send `noindex` (or sit behind a login). Default hosting addresses (for example `*.workers.dev`, `*.pages.dev`, `*.vercel.app`) are `noindex` or redirect to the real domain, so they don't compete with it.
- [ ] **HTTPS everywhere** and no mixed content.

## 1. Submit the site to Google Search Console (and Bing)

- [ ] Add a **Domain property** in [Google Search Console](https://search.google.com/search-console) and verify it with the DNS TXT record (covers every subdomain and protocol).
- [ ] Use **URL Inspection** on the home page and two key pages: "URL is on Google" or "Request indexing"; check the rendered HTML shows the real content.
- [ ] Add the site to [Bing Webmaster Tools](https://www.bing.com/webmasters) (you can import from Search Console). Bing's index feeds Bing, Microsoft Copilot and several other search and AI tools. Optionally turn on IndexNow so changes are picked up fast.
- [ ] Give access to a second owner so the account doesn't depend on one person.

## 2. Create and submit an XML sitemap

- [ ] `/sitemap.xml` lists every public, indexable, canonical URL, and nothing else: no private, redirected, `noindex` or duplicate URLs. Add `<lastmod>` only if it's accurate.
- [ ] `/robots.txt` allows crawling of public pages and points to the sitemap: `Sitemap: https://example.com/sitemap.xml`.
- [ ] Submit the sitemap in Search Console (**Sitemaps**) and Bing Webmaster Tools; check it reports "Success" and the discovered count matches.
- [ ] The sitemap is regenerated when pages are added or removed (generate it from the same list of pages the site uses, not by hand).

## 3. Write meta titles and descriptions

Every public page has its own:

- [ ] **Title** (`<title>`): unique, about 50 to 60 characters, the page's main topic first, brand last ("Team planner for small clinics · Brand").
- [ ] **Meta description:** unique, about 140 to 160 characters, says what the page offers and why to click. Google may rewrite it, but a good one is used more often.
- [ ] **One `<h1>`** that matches the page's topic, then a logical `h2`/`h3` outline.
- [ ] **Social preview tags:** `og:title`, `og:description`, `og:url`, `og:image` (1200 × 630 px), `og:type`, and `twitter:card` = `summary_large_image`. Test the preview with the platforms' share debuggers.
- [ ] `<html lang="…">` is set; multilingual sites use `hreflang` links.
- [ ] Favicon and app icons are in place (they show in search results).

## 4. Optimise for SEO and GEO

**GEO** (generative engine optimisation) means showing up and being cited in AI answers. Google's own guidance (May 2026) says AI features use the same index, so **good SEO is the foundation of GEO**. It says you don't need special "AI files", chunked content or special schema for its AI features, and that its spam policies now cover AI Overviews and AI Mode.

**Content**

- [ ] Each page answers a real question people search for, in plain words, with the answer early on the page and details below.
- [ ] Content is original and useful (your own data, examples, prices, comparisons), not rewritten from other sites. Thin AI-generated pages, or many near-identical pages for each keyword variation, can count as spam.
- [ ] Clear facts that are easy to quote and check: who you are, what you offer, prices, locations, dates, contact. Keep them identical everywhere (site, profiles, directories).
- [ ] An About page and real authors or company details build trust (this overlaps with the business-details item in the compliance checklist).
- [ ] Internal links with descriptive text connect related pages; no orphan pages.

**Technical**

- [ ] Core Web Vitals are good on mobile (check PageSpeed Insights and the Search Console report): fast largest paint, no layout shift, quick response to taps.
- [ ] Mobile-friendly and accessible (accessibility and SEO overlap: headings, alt text, link text).
- [ ] Images have descriptive file names and alt text, explicit width and height, and modern formats.
- [ ] Structured data (JSON-LD) where it fits: `Organization` (name, logo, `sameAs` social links), `WebSite`, and page types such as `Product`, `FAQPage`, `Article`, `LocalBusiness`, `BreadcrumbList`. Validate with Google's Rich Results Test. It earns rich results; it isn't a requirement for AI answers.

**AI answer engines beyond Google**

- [ ] Their search crawlers are allowed in `robots.txt` and in your CDN or bot settings if you want to be cited: for example `OAI-SearchBot` (ChatGPT search), `Claude-SearchBot` (Claude), `PerplexityBot`. Their *training* crawlers (such as `GPTBot`, `ClaudeBot`, `Google-Extended`, `CCBot`) are a separate choice: blocking training doesn't block search citations when the vendor separates the two.
- [ ] Optional: an `/llms.txt` summary for tools that read it. It's an unofficial proposal; Google says it doesn't use it, so treat it as a nice-to-have.
- [ ] Ask a few AI assistants your main questions after launch and note whether and how you're cited; fix wrong facts at the source (your site and profiles).

## 5. Create a Google Business Profile (if eligible)

A free listing that shows your business on Google Maps and in local results, with hours, reviews, photos and directions.

- [ ] **Check eligibility first.** Google only allows businesses that **make in-person contact with customers during stated hours**: a storefront or office customers visit, or a service-area business that goes to customers (plumber, cleaner, consultant on site). **Online-only businesses, brands and lead-generation sites are not eligible**, and neither are P.O. boxes or virtual mailboxes.
- [ ] If eligible: create it at [business.google.com](https://business.google.com), verify it, and fill in the exact legal name (no keyword stuffing), category, address or service area, hours, phone, website, photos and services. Keep name, address and phone identical to the website.
- [ ] Ask real customers for reviews and reply to them; never buy or fake reviews (see the compliance checklist's honest-design items).
- [ ] Also claim **Bing Places** and, where relevant, Apple Business Connect and local directories with the same details.
- [ ] **If not eligible (online-only):** skip the profile. Put the business details on the site, add `Organization` structured data, and keep consistent profiles on the platforms your customers use.

## 6. After launch: watch and fix

- [ ] Week 1: Search Console **Pages** report shows the public pages as indexed; fix anything under "Why pages aren't indexed" (blocked, `noindex`, soft 404, duplicate without canonical).
- [ ] Monthly: **Performance** report (queries, clicks, positions), Core Web Vitals, broken links and 404s, and sitemap status.
- [ ] Changing a URL? Add a 301 redirect from the old one and update internal links and the sitemap.
- [ ] Re-run this guide after a redesign, a domain change or a new section.

## Sign-off

| Step | Status | Evidence / note |
| ---- | ------ | --------------- |
| 0 Crawlable and private pages kept out | | |
| 1 Search Console and Bing verified | | |
| 2 Sitemap and robots.txt submitted | | |
| 3 Titles, descriptions, social tags | | |
| 4 SEO and GEO basics | | |
| 5 Google Business Profile (or "not eligible: online-only") | | |
| 6 Monitoring owner and schedule | | |

---

## Appendix: projects built with the project starter

Skip this in other projects. In starter projects the file lives at `docs/SEO_AND_DISCOVERABILITY.md`.

**What the starter does by default.** It's a single-page app on Cloudflare Workers: every URL returns the same `index.html` with one title and description, and unknown URLs return 200 (`"not_found_handling": "single-page-application"`). The design system also says the app has no landing page (`DESIGN_SYSTEM.md` §1.1). That's right for an invite-only app, which should simply stay out of search. For a product with public pages, do this:

1. **Default to private.** Add `<meta name="robots" content="noindex" />` to `index.html`, so every app screen and unknown URL stays out of search, and add a `public/_headers` rule so the `workers.dev` address is never indexed:

   ```txt
   https://:subdomain.workers.dev/*
     X-Robots-Tag: noindex
   ```

2. **Publish public pages through the Worker.** List them in one place, route them through the Worker, and let it swap in each page's meta tags (and drop the `noindex`) only in production. Add `"binding": "ASSETS"` and the page paths to `run_worker_first` in `wrangler.jsonc`, add `SITE_URL` (the canonical origin, for example `https://example.com`) to `vars` and `docs/ENV_AND_SECRETS.md`, then run `pnpm cf-typegen`.

   ```jsonc
   "assets": {
     "binding": "ASSETS",
     "not_found_handling": "single-page-application",
     "run_worker_first": ["/api/*", "/", "/pricing", "/about", "/robots.txt", "/sitemap.xml"]
   },
   ```

   ```ts
   // shared/seo.ts: the one list of public pages (the app can reuse it for useDocumentTitle)
   export type PublicPage = { title: string; description: string; image?: string }

   export const PUBLIC_PAGES: Record<string, PublicPage> = {
     "/": { title: "{{PROJECT_NAME}}: {{ONE_LINE_SUMMARY}}", description: "{{PROJECT_DESCRIPTION}}" },
     // "/pricing": { title: "Pricing · {{PROJECT_NAME}}", description: "…" },
   }
   ```

   ```ts
   // worker/seo.ts
   import { PUBLIC_PAGES } from "../shared/seo"

   const esc = (s: string) => s.replace(/[&<>"']/g, (c) => `&#${c.charCodeAt(0)};`)
   const isProduction = (env: Env) => env.ENVIRONMENT === "production"

   export async function seoResponse(request: Request, env: Env): Promise<Response | null> {
     const url = new URL(request.url)
     if (request.method !== "GET" && request.method !== "HEAD") return null

     if (url.pathname === "/robots.txt") {
       const body = isProduction(env)
         ? `User-agent: *\nAllow: /\nDisallow: /api/\n\nSitemap: ${env.SITE_URL}/sitemap.xml\n`
         : "User-agent: *\nDisallow: /\n"
       return new Response(body, { headers: { "Content-Type": "text/plain; charset=utf-8" } })
     }

     if (url.pathname === "/sitemap.xml") {
       const urls = Object.keys(PUBLIC_PAGES)
         .map((path) => `<url><loc>${esc(new URL(path, env.SITE_URL).toString())}</loc></url>`)
         .join("")
       const xml = `<?xml version="1.0" encoding="UTF-8"?><urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">${urls}</urlset>`
       return new Response(xml, { headers: { "Content-Type": "application/xml; charset=utf-8" } })
     }

     const page = PUBLIC_PAGES[url.pathname]
     if (!page) return null

     const shell = await env.ASSETS.fetch(new URL("/", url))
     const canonical = new URL(url.pathname, env.SITE_URL).toString()
     const image = new URL(page.image ?? "/brand/og.png", env.SITE_URL).toString()
     const tags = [
       `<link rel="canonical" href="${esc(canonical)}" />`,
       `<meta property="og:type" content="website" />`,
       `<meta property="og:url" content="${esc(canonical)}" />`,
       `<meta property="og:title" content="${esc(page.title)}" />`,
       `<meta property="og:description" content="${esc(page.description)}" />`,
       `<meta property="og:image" content="${esc(image)}" />`,
       `<meta name="twitter:card" content="summary_large_image" />`,
     ].join("")

     return new HTMLRewriter()
       .on('meta[name="robots"]', {
         element(el) {
           if (isProduction(env)) el.remove()
         },
       })
       .on("title", {
         element(el) {
           el.setInnerContent(page.title)
         },
       })
       .on('meta[name="description"]', {
         element(el) {
           el.setAttribute("content", page.description)
         },
       })
       .on("head", {
         element(el) {
           el.append(tags, { html: true })
         },
       })
       .transform(shell)
   }
   ```

   ```ts
   // worker/index.ts: keep the Hono API as `app`, and export a fetch that tries SEO routes first
   import { seoResponse } from "./seo"
   // …existing Hono app…
   export default {
     async fetch(request: Request, env: Env, ctx?: ExecutionContext) {
       return (await seoResponse(request, env)) ?? app.fetch(request, env, ctx)
     },
   }
   ```

   The existing tests call `app.fetch(request, env)` on the default export, and that still works. Add tests for `/robots.txt`, `/sitemap.xml` and one public page (canonical present, `noindex` kept outside production).

3. **Give public pages real text in the HTML.** The Worker above fixes meta tags, but the body is still rendered by React. That's enough for Google; AI crawlers need the text in the HTML too. For a few marketing pages, the simplest fix is prerendered static HTML (Workers serves `/pricing/index.html` for `/pricing` before the SPA fallback); otherwise inject a short server-rendered summary with `HTMLRewriter`. Decide this early (the second early decision).

4. **Public pages are a separate surface.** A landing page goes against `DESIGN_SYSTEM.md` §1.1 for the app itself, so the owner approves it and it's recorded in §22. Expressive visuals on it follow `DESIGN_INSPIRATION.md`, and must not hurt Core Web Vitals (lazy-load effects).

| Item | Who does it |
| --- | --- |
| Early decisions recorded in `docs/GUIDE.md` | `project-manager` with the owner |
| `noindex` default, Worker SEO routes, sitemap, robots, tests | `backend-developer` |
| Titles, descriptions, headings, copy, OG image | `product-designer` |
| Core Web Vitals, rendered-HTML and no-JS checks, Lighthouse | `qa-tester` |
| Cloudflare AI bot policy, custom domain, redirects, DNS verification | `devops` (owner approves) |
| Search Console, Bing, Business Profile accounts | the owner |
