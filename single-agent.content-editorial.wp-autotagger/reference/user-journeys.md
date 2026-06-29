# User journeys — wp-autotagger

## J1 — Submit a tech post and get tags applied

**Preconditions:** Service running on declared port (`http://localhost:9796/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9796/` → App UI tab.
2. Click the **tech** quick-fill link to populate the Post URL field with the seeded technology how-to post URL and set Tag style to `MIXED`.
3. Leave Max tags at the default (10). Enter a value in Submitted by.
4. Click **Submit for tagging**.

**Expected:**
- The new card appears in the live list with status `INGESTED` within 1 s.
- The card transitions to `BODY_FETCHED` within 1 s. The right-pane detail shows the first 500 chars of the fetched post body and the post metadata (title, author, published date).
- Within 30 s the card reaches `TAGS_APPLIED`. The right pane shows: between 3 and 10 tag chips ordered by confidence, each with a confidence bar. The rationale paragraph is non-empty.
- Status pill shows `TAGS_APPLIED` in green. Tag count badge shows the exact number of applied tags.

## J2 — Guardrail blocks an over-count tag list

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `propose-tags.json` includes a deliberately malformed entry whose tag count exceeds `maxTags`.

**Steps:**
1. Set Max tags to 5.
2. Submit the seeded tech post three times in a row (J1 steps × 3).
3. Watch the third submission's lifecycle in the network panel (`/api/posts/sse`).

**Expected:**
- The third submission's first agent iteration produces an `applyTags` tool call with more than 5 tags.
- The `before-tool-call` guardrail rejects it with a structured error naming the count check. No tags are written to the stub.
- The agent loop retries on iteration 2 (and 3 if needed) with a trimmed tag list of 5 or fewer. The card reaches `TAGS_APPLIED` with a proposal whose tag count is ≤ 5.
- The service log shows one `guardrail.reject` line per rejected tool call with the structured-error code `TAG_COUNT_EXCEEDED`.

## J3 — Guardrail blocks a wrong-post-id tool call

**Preconditions:** Mock LLM mode. The mock includes an entry where the `applyTags` tool call carries a `postId` that does not match the active job's resolved post ID.

**Steps:**
1. Submit the seeded product-review post.
2. Watch the lifecycle in the network panel.

**Expected:**
- The agent's first `applyTags` call carries an incorrect `postId`.
- The guardrail rejects it with structured error code `POST_ID_MISMATCH`. The wrong post receives no tags.
- The agent corrects the `postId` on the next iteration and re-calls successfully.
- The card reaches `TAGS_APPLIED` and the right-pane detail shows the correct post's tags.

## J4 — Credential token never reaches the LLM

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the body-redaction runs either way).

**Steps:**
1. Manually POST to `/api/posts` with a custom `postUrl` that resolves to a body containing the literal string `wp-app-password: ABC123xyz` (simulate a carelessly-included credential in a draft post).
2. Wait for `TAGS_APPLIED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/posts/{id}` and read `body.body`.

**Expected:**
- The logged LLM call body contains `[REDACTED-TOKEN]` in place of `ABC123xyz`. The raw credential string does not appear anywhere in the agent call chain.
- `body.body` in the JSON still contains the raw text — the entity's audit log preserves it.

## J5 — Multi-style tagging produces distinct results

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the same news summary post URL three times, each time with a different `tagStyle` (DESCRIPTIVE, KEYWORD_DENSE, MIXED).
2. Wait for all three to reach `TAGS_APPLIED`.

**Expected:**
- The three tag proposals differ in character: DESCRIPTIVE tags read as noun phrases, KEYWORD_DENSE tags are single terms or short compound terms, MIXED blends both.
- All three proposals pass the guardrail (non-empty, within maxTags, valid characters).
- Each proposal's `tagStyle` field matches the style requested in the submission.

## J6 — Failed job retains partial state

**Preconditions:** Mock LLM mode with all 3 iterations returning rejected tool calls (can be simulated by setting `maxTags = 1` and using a mock entry that always returns 2+ tags).

**Steps:**
1. Set Max tags to 1.
2. Submit any seeded post.
3. Wait for the status to reach `FAILED`.

**Expected:**
- The card shows `FAILED` status in red.
- The right-pane detail still shows the fetched post body and metadata (the `BODY_FETCHED` event persisted before the agent failed).
- No partial tag proposal is shown — the `proposal` field is null.
- The service log shows three `guardrail.reject` lines followed by the workflow's `error` step transition.
