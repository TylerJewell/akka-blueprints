# User journeys — story-teller

## J1 — Submit a seeded prompt and get a story

**Preconditions:** Service running on declared port (`http://localhost:9549/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9549/` → App UI tab.
2. From the **Genre** dropdown, pick `Mystery`.
3. Click **Load seeded example** to fill the prompt textarea with the mystery investigation seed.
4. Leave style constraints at their defaults (NEUTRAL tone, 300 words, THIRD person).
5. Click **Generate story**.

**Expected:**
- The new card appears in the live list with status `REQUESTED` within 1 s.
- The card transitions to `ENRICHED` within 1 s. The right-pane detail shows the enriched prompt and content tag chips (e.g., `mystery`, `investigation`).
- Within 30 s the card reaches `STORY_RECORDED`. The right pane shows: a non-empty title, a narrative body of at least 150 characters with at least one paragraph break, and an author's note of at least 30 characters. The genre badge reads `Mystery`.
- Within 1 s of `STORY_RECORDED`, the card reaches `SCORED` and shows a quality score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a malformed story

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `generate-story.json` includes deliberately malformed entries (one with an empty body; one with a genre value not in the allowed set).

**Steps:**
1. Submit any seeded prompt three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/stories/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed story.
- The `before-agent-response` guardrail rejects it. The malformed story NEVER lands in `StoryEntity` — there is no `StoryRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed story. The card transitions to `STORY_RECORDED` with a story that satisfies all five guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J3 — Quality scorer flags a thin story

**Preconditions:** Mock LLM mode. The mock's `generate-story.json` contains a "thin" entry with a body that is exactly 150 characters (minimum), a duplicate of the title in the author's note, and no paragraph break.

**Steps:**
1. Submit any seeded prompt and observe which mock entry the seeder selects (check the service log for the mock seed value).
2. If the thin entry is not selected on first try, submit a second prompt with the same genre.

**Expected:**
- The story lands well-formed (the guardrail checks structural validity only, not quality).
- The quality score chip shows **1** or **2** and the rationale names the specific failures (e.g., "Body meets minimum length but has no paragraph breaks; author's note duplicates title.").
- The card's border highlights red. The user sees the flag immediately in the live list.

## J4 — Unsafe prompt is blocked before agent invocation

**Preconditions:** Service running (any model provider — the safety filter runs before the agent).

**Steps:**
1. In the App UI, type a prompt containing a phrase in the disallowed content blocklist (e.g., a phrase matching the `explicit-violence` keyword set).
2. Click **Generate story**.

**Expected:**
- The card appears with status `REQUESTED` then immediately transitions to `BLOCKED` (within 1 s).
- The right-pane detail shows the block reason: "Prompt contains disallowed content category: explicit-violence" (or whichever category matched).
- No `GenerationStarted` event appears in the SSE stream — the agent is never called.
- The service log shows a `prompt.blocked` entry with the story id and the matched category. No LLM API call is made.

## J5 — Style constraints are honoured

**Preconditions:** Service running with a real model provider.

**Steps:**
1. Submit the fantasy quest seed with: tone = `WHIMSICAL`, wordCountTarget = `100`, pointOfView = `FIRST`.
2. Submit the same prompt again with: tone = `GRITTY`, wordCountTarget = `600`, pointOfView = `THIRD`.
3. Wait for both to reach `SCORED`.

**Expected:**
- The first story is noticeably lighter in register than the second (whimsy vs. weight in word choice and sentence structure).
- The first story's body is approximately 100 words (within ±20%); the second is approximately 600 words.
- The first story uses first-person pronouns; the second uses third-person throughout.
- Both stories receive a quality score ≥ 3 (they are well-formed and meet the word count target).
