# User journeys — social-amplifier

## J1 — Submit an article and get posts published across all platforms

**Preconditions:** Service running on declared port (`http://localhost:9860/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded article `Akka 3.6 launch post` has a matching `src/main/resources/sample-data/articles/akka-3-6-launch.json` file.

**Steps:**
1. Open `http://localhost:9860/` → App UI tab.
2. From the **Pick a seeded article** dropdown, pick `Akka 3.6 launch post`.
3. Confirm all three platforms (LinkedIn, X, Bluesky) are checked.
4. Click **Amplify**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `PARSING` within 1 s more.
- Within ~20 s the card reaches `PARSED`. The right pane shows the Parsed article panel with a headline and ≥ 3 key messages; each message has a non-empty `text` and a `platformHint`.
- Within ~20 s more the card reaches `DRAFTED`. The right pane shows three draft cards, one per platform; each card shows the draft text, hashtags, and a green brand-check badge. All character counts are within their platform limits.
- Within ~20 s more the card reaches `PUBLISHED`. The right pane shows three publication receipts, one per platform; each receipt has a non-empty `postUrlStub` and a `publishedAt` timestamp. The brand-audit score chip shows ≥ 4/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Brand-policy response guardrail rejects an over-length X draft

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `draft-posts.json` includes one entry whose X `Draft` has `characterCount > 280` — the deliberately brand-policy-violating entry.

**Steps:**
1. Submit any seeded article three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/amplifications/sse`).

**Expected:**
- On the third submission's `draftStep`, the agent's first iteration returns a `DraftSet` where the X draft has `characterCount > 280`. `BrandPolicyGuardrail`'s response hook rejects the response; a `BrandCheckFailed{platform: "X", rule: "character-limit", reason: "brand-policy-violation: character-limit: ..."}` event lands on the entity.
- The over-length draft is NEVER written to `DraftsProduced` — there is no `DraftsProduced` event in the entity log for that iteration.
- The agent's second iteration returns a corrected X draft with `characterCount ≤ 280`. The `brandCheckAttempts` field on the corrected X draft is 1. The run eventually reaches `PUBLISHED` as in J1.
- The card in the App UI shows the small orange dot indicating a brand-check rejection fired. The brand-check-rejection log strip on the right pane shows the one rejected draft with its rule, reason, and timestamp.

## J3 — Publish-intent tool-call guardrail blocks a premature publish call

**Preconditions:** Service running with the mock LLM selected. The mock's `publish-posts.json` includes one entry whose `tool_calls` array starts with `publishToLinkedIn` before `DraftsProduced` has been recorded — the deliberately tool-gated entry.

**Steps:**
1. Submit any seeded article. (The tool-gated mock entry fires on the first PUBLISH task iteration of every 5th run by the mock's `seedFor(amplificationId)` modulo. Run five amplifications to reproduce.)
2. Watch for a card that shows a small orange dot alongside the `PUBLISHED` status.

**Expected:**
- On the matching run's `publishStep`, the agent's first iteration calls `publishToLinkedIn` while `AmplificationEntity.status = DRAFTING`. `BrandPolicyGuardrail`'s tool-call hook rejects the call; a `PublishToolBlocked{platform: "LINKEDIN", tool: "publishToLinkedIn", reason: "publish-gated: ..."}` event lands on the entity.
- The publish stub is NEVER reached — there is no log line from `PublishTools.publishToLinkedIn` body.
- The agent's second iteration runs the correct PUBLISH sequence after the entity confirms `DraftsProduced`. The run eventually reaches `PUBLISHED`.
- The publish-tool-blocked log strip on the right pane shows the one blocked call with its tool, reason, and timestamp.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider.

**Steps:**
1. Submit any seeded article.
2. Wait for `PUBLISHED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the `amplificationId`.

**Expected:**
- The PARSE task's log entries show only `fetchArticle` and `extractKeyMessages` calls.
- The DRAFT task's log entries show only `draftPost` and `applyHashtags` calls.
- The PUBLISH task's log entries show only `publishToLinkedIn`, `publishToX`, and `publishToBluesky` calls.
- No cross-phase calls appear. If the brand-check or publish-gate path fires, a single `guardrail.reject` line precedes the agent's retry; the rejected call is logged but the tool body is never executed.
- The order of tasks in the log is PARSE → DRAFT → PUBLISH. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Platform count parity

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded article `Q3 developer survey results` with all three platforms selected.
2. Wait for `PUBLISHED`.

**Expected:**
- `DraftSet.drafts.length == 3` and `PublishedSet.receipts.length == 3`.
- Every `draft.platform` matches a `targetPlatform` from the request; no platform is duplicated or missing.
- Every `receipt.platform` matches a `draft.platform` in the `DraftSet`, one-to-one.
- If the agent produced 2 drafts instead of 3 (silent platform collapse), the `BrandPolicyScorer` would flag "receipt coverage failure" and score ≤ 3. A score of 4 or 5 with rationale containing "all checks passed" confirms the parity held.

## J6 — Article URL with no matching sample file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/articles/<custom-slug>.json` exists for the typed URL.

**Steps:**
1. In the App UI, paste a custom URL (e.g. `https://example.com/my-blog-post`) into the article URL field.
2. Click **Amplify**.

**Expected:**
- `ParseTools.fetchArticle` returns an empty string; `extractKeyMessages` returns `[]`.
- The agent's PARSE task returns a `ParsedArticle` with `keyMessages = []`.
- The workflow advances to `draftStep`. The DRAFT task returns a `DraftSet` with `drafts = []` (the agent's refusal behaviour per `prompts/amplifier-agent.md`).
- The workflow advances to `publishStep`. The PUBLISH task returns a `PublishedSet` with `receipts = []`.
- The brand-audit score chip shows 1 (receipt coverage = 0 receipts for 0 drafts passes trivially, but no other check has material to score — rationale reads "no drafts to evaluate"). The run reaches `PUBLISHED`; nothing crashes; the empty result is honestly empty.
