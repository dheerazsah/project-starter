# AI features

How to build AI features people can trust: answers only from the app's own sources, with citations; structured output that is validated; per-person quotas with refunds; quality-first model choice; and cost controls. Patterns from a production knowledge app (grounded Claude answers, AI drafts from uploaded documents) and a small Workers AI feature, generalised. Read the `claude-api` skill before writing Claude code: it has the current model ids, parameters and pricing. Security rules for AI are in [`SECURITY.md` §7](./SECURITY.md#7-ai-prompt-injection-and-data-exposure).

## Contents
1. [Principles](#1-principles)
2. [Grounded answers with citations](#2-grounded-answers-with-citations)
3. [Quotas, refunds and spend limits](#3-quotas-refunds-and-spend-limits)
4. [Structured output](#4-structured-output)
5. [Small talk and questions with nothing to search](#5-small-talk-and-questions-with-nothing-to-search)
6. [Model choice](#6-model-choice)
7. [Cost controls](#7-cost-controls)
8. [Streaming and the UI](#8-streaming-and-the-ui)
9. [Testing AI features](#9-testing-ai-features)

## 1. Principles

- **Answers only from sources.** The model writes from passages the app retrieved and the caller may see. No outside knowledge for company facts. If the sources don't cover it, say so plainly and point to who might know.
- **Every claim is traceable:** citations link to the source and quote the passage.
- **The person stays in charge:** AI drafts are proposals the person reviews before saving; nothing is published by the model.
- **It degrades, never breaks:** with no key, a quota hit or a failed call, the feature falls back (quote the best passage, "Write it yourself") and says so.
- **Quality first, waste second:** pick the model that gets it right; cut cost with retrieval, caching, quotas and short prompts, not with a weaker model on work people rely on.
- **Untrusted input stays data:** documents and questions never become instructions ([`SECURITY.md` §7](./SECURITY.md#7-ai-prompt-injection-and-data-exposure)).

## 2. Grounded answers with citations

Flow: **question → visibility-filtered search (D1 FTS5) → top passages → Claude with each passage as a citable document → answer with `[n]` markers → stored with its citations.**

```text
POST /api/ask
  1. reserve one ask from the person's quota (429 + Retry-After when over)
  2. passages = search(question, visibleTo(user))        // SQL filter BEFORE ranking and LIMIT
  3. passages.length === 0 → "no approved source" answer  // no model call, refund the ask
  4. no API key → quote the best passage                  // degrade, still cited
  5. Claude: system prompt + last 3 turns + passages as documents (citations on) + question
  6. build the answer: text + [n] after cited text, one ordinal per source
  7. store message + citations + every source version used, in one db.batch
```

**System prompt** (short and stable; it says what to do when sources don't answer):

```ts
const SYSTEM = `You answer questions from {{ORG}} employees using only the company sources provided with each question.
If the sources don't answer the question, say plainly that {{PROJECT_NAME}} has no approved source on that, and suggest who might know (the source owner, if one is relevant). Never guess, and never use outside knowledge for {{ORG}} policy or practice.
Be concise and use plain language. Format only with short paragraphs, "- " or "1." lists, **bold**, and at most short "## " headings. No links, tables, italics or block quotes.
Don't talk about "documents" or how you were given them; just answer.`

export const NO_SOURCE_ANSWER = "There's no approved source that covers this yet. Try different words, or ask the team that owns this topic."
const REFUSAL_ANSWER = "This can't be answered here. Try asking it a different way, or ask the team that owns this topic."
```

**The request:** passages as `document` blocks with citations enabled, earlier turns as plain history (markers stripped):

```ts
function buildRequest(env: Env, question: string, history: Turn[], passages: Passage[]) {
  const messages: Anthropic.Beta.BetaMessageParam[] = []
  for (const turn of history.slice(-3)) {
    messages.push({ role: "user", content: turn.question })
    messages.push({ role: "assistant", content: turn.answer.replace(/\s?\[\d+\]/g, "") })
  }
  messages.push({
    role: "user",
    content: [
      ...passages.map((p): Anthropic.Beta.BetaRequestDocumentBlock => ({
        type: "document",
        source: { type: "text", media_type: "text/plain", data: p.text },
        title: p.heading ? `${p.title} — ${p.heading}` : p.title,
        citations: { enabled: true },
      })),
      { type: "text", text: question },
    ],
  })
  return {
    model: env.ANTHROPIC_MODEL,            // a var, so it can change without a deploy of code
    thinking: { type: "adaptive" as const },
    output_config: { effort: "low" as const }, // short factual answers; raise for harder tasks
    system: SYSTEM,
    messages,
  }
}
```

**Building the answer from citations** (one ordinal per source, markers after the cited text, refusals and truncation handled):

```ts
export function resultFromMessage(response: Anthropic.Beta.BetaMessage, passages: Passage[]): AnswerResult {
  if (response.stop_reason === "refusal") return { answer: REFUSAL_ANSWER, mode: "none", citations: [] }
  const ordinals = new Map<string, number>() // sourceId -> ordinal
  const citations: AnswerCitation[] = []
  let answer = ""
  for (const block of response.content) {
    if (block.type !== "text") continue // skip thinking blocks
    const marks = new Set<number>()
    for (const cite of block.citations ?? []) {
      if (cite.type !== "char_location") continue
      const passage = passages[cite.document_index]
      if (!passage) continue
      let ordinal = ordinals.get(passage.sourceId)
      if (ordinal === undefined) {
        ordinal = citations.length + 1
        ordinals.set(passage.sourceId, ordinal)
        citations.push({ ordinal, sourceId: passage.sourceId, versionId: passage.versionId, quote: shorten(cite.cited_text, 300) })
      }
      marks.add(ordinal)
    }
    if (marks.size === 0) { answer += block.text; continue }
    const tail = /\s*$/.exec(block.text)?.[0] ?? ""
    answer += block.text.slice(0, block.text.length - tail.length) + [...marks].sort((a, b) => a - b).map((n) => ` [${n}]`).join("") + tail
  }
  answer = answer.trim()
  if (!answer) return quoteAnswer(passages) // nothing usable: quote the best passage
  if (response.stop_reason === "max_tokens") answer += "\n\n(This answer was cut short. Try asking a narrower question.)"
  return { answer, mode: "ai", citations }
}

/** AI off or failed: quote the best passage so the answer still helps, cited. */
export function quoteAnswer(passages: Passage[]): AnswerResult {
  const top = passages[0]
  return { answer: `${shorten(top.text, 1200)} [1]`, mode: "quote", citations: [{ ordinal: 1, sourceId: top.sourceId, versionId: top.versionId, quote: shorten(top.text, 300) }] }
}
```

On any API error: log the kind (`RateLimitError`, `APIConnectionError`, status), never the content, and fall back to `quoteAnswer`. Store which mode answered (`ai`, `quote`, `none`) so reports and costs can be counted.

**Retrieval quality matters more than the prompt.** Weight titles above body (`bm25(chunks, 4.0, 2.0, 1.0)`), chunk sources by heading, require a minimum relevance before a passage is used, and keep a small topic-synonym list for the domain's words. A "Report this answer" action feeds fixes back to the source owner.

## 3. Quotas, refunds and spend limits

Every paid call takes a unit from the person's quota **before** any paid work, in **one statement** that checks and increments (parallel requests can't pass the limit together). If the call then fails **without being billed**, the unit is refunded.

```ts
export const DRAFTS_PER_HOUR = 10

export async function reserveDraft(db: D1Database, user: CurrentUser) {
  const at = now()
  const windowOpen = new Date(Date.now() - 3_600_000).toISOString()
  const result = await db.prepare(
    `INSERT INTO draft_quota (user_id, window_start, count) VALUES (?1, ?2, 1)
     ON CONFLICT (user_id) DO UPDATE SET
       count = CASE WHEN draft_quota.window_start <= ?3 THEN 1 ELSE draft_quota.count + 1 END,
       window_start = CASE WHEN draft_quota.window_start <= ?3 THEN ?2 ELSE draft_quota.window_start END
     WHERE draft_quota.window_start <= ?3 OR draft_quota.count < ?4`,
  ).bind(user.id, at, windowOpen, DRAFTS_PER_HOUR).run()
  if (result.meta.changes === 0) {
    tooMany(
      `You've used AI drafts ${DRAFTS_PER_HOUR} times in the last hour. Try again later, or write it yourself.`,
      await quotaWindowStart(db, "draft_quota", user.id),
      3_600_000,
    )
  }
}

export const refundDraft = (db: D1Database, user: CurrentUser) =>
  db.prepare("UPDATE draft_quota SET count = MAX(count - 1, 0) WHERE user_id = ?").bind(user.id).run()

/**
 * Did this failed call cost anything? Not when the request never got a response
 * (connection refused) or the API answered with an HTTP error before streaming
 * (a numeric status: 429, 5xx, 400). A timeout or a mid-stream error may have been billed.
 */
export function billedBy(error: unknown) {
  if (error instanceof Anthropic.APIConnectionTimeoutError) return true
  if (error instanceof Anthropic.APIConnectionError) return false
  if (error instanceof Anthropic.APIError) return typeof error.status !== "number"
  return true
}

// In the route:
await reserveDraft(db, user)
const usage = { billed: false }
try {
  const draft = await draftFromDocument(env, text, usage) // sets usage.billed once Claude answered
  return c.json(draft)
} catch (error) {
  if (!usage.billed) await refundDraft(db, user)
  throw error
}
```

Rules:
- Free work is never counted: a cached result, a question with no sources, small talk ([§5](#5-small-talk-and-questions-with-nothing-to-search)), a request refused before the call (too long, wrong type).
- Refuse expensive input **before** paying: a PDF over N pages, a document over N characters ("Use a shorter file, or split it").
- Quota messages say the limit, when it frees up (`Retry-After`) and the manual alternative.
- Set a **monthly spend limit** in the Anthropic console, and record it in `ROADMAP.md` risks.

## 4. Structured output

When the model fills a form (a draft from an uploaded document, an extraction), ask for **JSON that matches a schema**, then validate it again with zod and sanitise each field.

```ts
const OUTPUT_SCHEMA = {
  type: "object",
  additionalProperties: false,
  required: ["title", "summary", "category", "reviewBy", "warnings", "body"],
  properties: {
    title: { type: "string" },
    summary: { type: "string" },
    category: { type: "string", enum: [...CATEGORIES] },
    reviewBy: { type: "string", description: "YYYY-MM-DD" },
    warnings: { type: "array", items: { type: "string" } },
    body: { type: "string" },
  },
}
const outputSchema = z.object({ title: z.string(), summary: z.string(), category: z.string(), reviewBy: z.string(), warnings: z.array(z.string()), body: z.string() })

const message = await client.beta.messages.create({
  model: env.ANTHROPIC_MODEL,
  max_tokens: 64000,
  thinking: { type: "adaptive" },
  output_config: { effort: "medium", format: { type: "json_schema", schema: OUTPUT_SCHEMA } },
  system: DRAFT_SYSTEM,
  messages: [{ role: "user", content: [{ type: "text", text: `Today is ${today}.\n\n<document>\n${documentText}\n</document>` }] }],
})
usage.billed = true // from here every outcome (refusal, cut off, bad JSON) was paid for
if (message.stop_reason === "refusal") fail(422, "The AI couldn't draft this document. Write it yourself instead.")
if (message.stop_reason === "max_tokens") console.error("Draft hit max_tokens")
const raw = outputSchema.parse(JSON.parse(textOf(message)))
```

Then **sanitise every field** rather than trusting it:
- Text fields trimmed and cut to the form's limits (at a paragraph or word boundary).
- Enums mapped to a known value or a default.
- Dates parsed and kept within a sane range (`tomorrow … +5 years`, else a default).
- A URL kept only if it's `https://` **and appears in the document itself** (no invented links).
- `warnings` shown to the person above the form ("The document contains instructions aimed at AI tools", "Pages 12–30 were left out").

**The drafting prompt** states the job, the audience, the review step, the injection rule ([`SECURITY.md` §7](./SECURITY.md#7-ai-prompt-injection-and-data-exposure)), faithfulness ("Use only what the document states. Never add facts, numbers, names, dates, links, advice or conclusions that aren't in it"), language ("Keep the document's language"), the allowed formatting, size limits, and what each output field means. Keep it stable (no per-request data in the system prompt) so it can be cached.

## 5. Small talk and questions with nothing to search

People type "hi", "thanks", "what can you do?" or a single generic word. Don't search, don't call the model, don't charge the quota:

```ts
// After stopwords and generic words are removed, nothing specific is left: it's small talk or too vague.
const parsed = parseQuery(question) // returns null when no specific terms remain
if (!parsed) {
  await refundAsk(db, user) // or check before reserving
  return reply(isGreeting(question) ? SMALL_TALK_ANSWER : TOO_VAGUE_ANSWER)
}

const SMALL_TALK_ANSWER = "Hi. Ask me about {{TOPICS}}, like “How do I submit travel expenses?”, and I'll answer from approved sources."
const TOO_VAGUE_ANSWER = "Could you say a bit more? For example, “What's the policy for working from abroad?”"
```

- The reply is friendly and short, says what the assistant can do, and gives one example question.
- It's stored like any answer (mode `none`, no citations) so the conversation reads naturally.
- Out-of-scope questions with search terms ("write me a poem") go through retrieval: no passages → `NO_SOURCE_ANSWER`, no model call.
- Suggested questions on the empty Ask screen teach the scope before anyone has to guess.

## 6. Model choice

Quality first. Choose by what the output is used for, not by price:

| Use | Model | Effort / thinking |
| --- | --- | --- |
| Answers people act on, drafting from documents, extraction into forms, reading PDFs and images | The strongest current Claude model (id in `ANTHROPIC_MODEL`; see the `claude-api` skill) | Adaptive thinking; `effort` low for short grounded answers, medium for drafting |
| Short deterministic transforms (transliteration, tagging, classification), high volume, low stakes | Workers AI or a small Claude model | `temperature: 0`, small `max_tokens`, cached |
| Embeddings for semantic search | Workers AI embeddings + Vectorize | n/a |

- Keep the model id in a var so it can change without code changes; record changes in `ROADMAP.md` decisions.
- Evaluate a model change on a fixed set of real questions with known good answers (see [§9](#9-testing-ai-features)) before switching.
- When in doubt, use the stronger model: a wrong answer costs more than the tokens saved.

## 7. Cost controls

- **Retrieve less, better:** send the top few relevant passages (with a relevance floor), not whole documents; cap document input (e.g. ~90,000 characters) and pages.
- **History:** the last 3 turns, markers stripped.
- **Cache** deterministic results: extracted file text in D1 (`file_texts`), Workers AI answers in the Cache API keyed by a prompt version.
- **Stable system prompts** (no per-request data) so prompt caching can help when requests repeat; put per-request data in the user turn.
- **Quotas per person** ([§3](#3-quotas-refunds-and-spend-limits)) and a **monthly spend limit** in the console.
- **Refuse early:** validate size, type and length before any paid call.
- **Timeouts and one retry:** `new Anthropic({ apiKey, maxRetries: 1, timeout: 120_000 })`.
- **Measure:** store `mode` (ai, quote, none) per answer and count paid calls per day; review in the roadmap.

## 8. Streaming and the UI

- Stream long answers (Server-Sent Events from the Worker: `text/event-stream`) so the first words appear within the Doherty threshold. Do every check (auth, quota, visibility, validation) **before** the stream opens, so errors are normal JSON responses.
- Stream text deltas only; when the final message arrives, replace the streamed text with the stored answer (which has the citation markers).
- The UI shows a quiet "Thinking…" state after 250 ms, a Stop button, citations as numbered chips that open the source at the quoted passage, and "Report this answer".
- Label AI drafts as drafts, show the model's `warnings` above the form, and never save without the person's action.

## 9. Testing AI features

- Swap the client with a stub in tests (`setTestClaudeClient`), only honoured when `ENVIRONMENT` is `test` ([`TESTING.md`](./TESTING.md)).
- Test: no passages → no call and `NO_SOURCE_ANSWER`; no key → quote mode; a refusal; `max_tokens`; citations mapped to the right sources and ordinals; invisible sources never reach the stub (assert on the request the stub received); quota exhausted → 429 with `Retry-After`; refund when the call fails unbilled, no refund when billed; malformed JSON from the model → a clear error, quota kept.
- Keep a small evaluation set (10–30 real questions with expected sources) and run it by hand before changing the model, prompt or retrieval.
