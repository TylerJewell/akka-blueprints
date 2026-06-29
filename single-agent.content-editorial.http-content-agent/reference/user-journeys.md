# User journeys — http-content-agent

## J1 — Submit a blog-post brief and get approved content

**Preconditions:** Service running on declared port (`http://localhost:9657/`); a valid
model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9657/` → App UI tab.
2. Click **Load seeded brief** and pick the "technology trends" brief (topic: *How event-driven
   architecture reduces operational overhead*, format: `BLOG_POST`, word count hint: 600).
3. Click **Generate**.

**Expected:**
- A new card appears in the live list with status `REQUESTED` within 1 s.
- The card transitions to `GENERATING` within 1 s.
- Within 30 s the card reaches `APPROVED`. The right pane shows a non-empty `title` in heading
  style, a `body` of ≥ 100 words, format badge `BLOG_POST`, and a word-count badge showing a
  count near 600.
- No rejected-draft text appears in the UI at any point.

## J2 — Guardrail blocks a prohibited-word draft

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The
mock's `generate-content.json` includes a deliberately malformed entry whose body contains a
prohibited phrase from `prohibited-words.txt`.

**Steps:**
1. Submit any seeded brief three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools
   (`/api/jobs/sse`).

**Expected:**
- The third submission's first agent iteration produces a draft containing a prohibited phrase.
- The `before-agent-response` guardrail rejects it. The rejected draft NEVER lands in
  `ContentJobEntity` — there is no `ContentApproved` event with the prohibited content.
- The agent loop retries on iteration 2 and produces a clean draft. The card transitions to
  `APPROVED` with content that passes all four guardrail checks.
- The service log shows one `guardrail.reject` line for the first iteration naming
  `prohibited-word` as the failed check.

## J3 — Guardrail blocks an incomplete draft (missing title)

**Preconditions:** Mock LLM mode. The mock's `generate-content.json` includes an entry with a
null `title` field. The mock selects this entry on the first iteration of every 3rd job.

**Steps:**
1. Submit a social-post brief (or any brief on a job that is the 3rd submission mod 3).

**Expected:**
- The first agent iteration returns a `GeneratedContent` with `title = null`.
- The guardrail rejects it with check name `missing-title`.
- The second iteration returns a well-formed draft with a non-empty `title`. The card transitions
  to `APPROVED`.
- The incomplete draft is never visible in the UI.

## J4 — Parallel jobs remain isolated

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit four briefs in quick succession (use different topics and output formats).
2. Wait for all four to reach `APPROVED` or `FAILED`.

**Expected:**
- Each job card shows its own topic, format badge, and generated content.
- No content from one job appears in another job's detail pane.
- Each job's `ContentGeneratorAgent` instance id is `"generator-" + jobId`, confirming per-job
  isolation in the service log.
- All four jobs complete independently; none blocks another.

## J5 — Social post respects word-count hint

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Load the "social campaign" seeded brief (format: `SOCIAL_POST`, word count hint: 120).
2. Click **Generate** and wait for `APPROVED`.

**Expected:**
- The approved content's `wordCount` falls between 60 and 180 (±50 % of 120, which is the
  guardrail tolerance).
- The format badge shows `SOCIAL_POST`.
- The body is a single cohesive social-media-appropriate paragraph with no internal headers.
