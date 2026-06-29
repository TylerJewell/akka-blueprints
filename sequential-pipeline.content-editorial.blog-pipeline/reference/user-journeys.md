# User journeys — AI Blog Writer Pipeline with Ollama

## J1 — Submit a topic and get a published post

**Preconditions:** Service running on declared port (`http://localhost:9628/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded topic `Getting started with reactive systems` has a matching `src/main/resources/sample-data/references/getting-started-with-reactive-systems.json` file.

**Steps:**
1. Open `http://localhost:9628/` → App UI tab.
2. From the **Pick a seeded topic** dropdown, pick `Getting started with reactive systems`.
3. Select post type `tutorial`.
4. Click **Run pipeline**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `RESEARCHING` within 1 s more.
- Within ~25 s the card reaches `RESEARCHED`. The right pane shows the Research-notes table with ≥ 4 rows; each row has a non-empty source, url, and summary.
- Within ~25 s more the card reaches `OUTLINED`. The right pane shows ≥ 3 outline sections; each section has a heading and ≥ 2 key points.
- Within ~25 s more the card reaches `DRAFTED`. The right pane shows draft prose for each section.
- Within ~25 s more the card reaches `EDITED`. The right pane shows edited prose with readability score chips per section.
- Within ~25 s more the card reaches `PUBLISHED`. The right pane shows the final `Post` with title, summary, sections, and reference chips; the policy-check pill shows green. At least one code fence appears in a section body (satisfying the tutorial post-type alignment rule).
- Total elapsed time: ≤ 90 s on the happy path.

## J2 — Content-policy guardrail blocks a prohibited-keyword draft

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `draft-post.json` includes one entry whose prose contains a prohibited keyword from `policy/prohibited-patterns.json` — this is the deliberately policy-violating entry.

**Steps:**
1. Submit any seeded topic three times in a row (J1 steps × 3), choosing any post type.
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/posts/sse`).

**Expected:**
- On the third submission's `draftStep`, the agent's first iteration produces prose with the prohibited keyword. `ContentPolicyGuardrail` blocks the response; a `GuardrailBlocked{phase: "DRAFT", reason: "policy-violation: prohibited keyword '...' found in prose"}` event lands on the entity.
- The non-compliant prose NEVER reaches `PostEntity.DraftWritten` — there is no event for that iteration.
- The agent's second iteration produces clean prose. The card eventually reaches `PUBLISHED` as in J1.
- The card in the App UI shows the small red dot indicating a block fired. The block-log strip on the right pane shows the one blocked response with its full structured reason.

## J3 — Post-type alignment check blocks a tutorial with no code fence

**Preconditions:** Mock LLM mode. The mock's `publish-post.json` includes one entry whose tutorial-type post contains no code fence in any section body.

**Steps:**
1. Submit a seeded topic with post type `tutorial` six times. (The post-type misalignment entry is selected once in every six runs by the mock's `seedFor(postId)` modulo.)
2. Watch the live list until a post card's policy-check pill shows red.

**Expected:**
- The flagged post's `publishStep` iteration 1 produces a `Post` with no code fence. `ContentPolicyGuardrail` blocks it with reason `"policy-violation: tutorial post must contain at least one code fence"`.
- The agent's second iteration produces a corrected `Post` with at least one code fence.
- The final post lands in `PUBLISHED` with the policy-check pill green.
- The block-log strip shows the one post-type alignment block.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded topic with any post type.
2. Wait for `PUBLISHED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the postId.

**Expected:**
- The RESEARCH task's log entries show only `searchReferences` and `fetchSummary` calls.
- The OUTLINE task's log entries show only `structureSections` and `expandKeyPoints` calls.
- The DRAFT task's log entries show only `writeParagraph` and `composeTile` calls.
- The EDIT task's log entries show only `applyToneAdjustments` and `checkReadability` calls.
- The PUBLISH task's log entries show only `formatPost` and `collectReferences` calls.
- No cross-phase calls appear. If a mock violation path fires, a `guardrail.block` line precedes the agent's retry; the blocked response is logged but never committed.
- The order of tasks in the log is RESEARCH → OUTLINE → DRAFT → EDIT → PUBLISH. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Opinion post-type alignment enforced end-to-end

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded topic `How event sourcing simplifies audit trails` with post type `opinion`.
2. Wait for `PUBLISHED`.

**Expected:**
- The published `Post` contains first-person stance markers ("I believe", "In my view", or "My take") in at least one section body (satisfying the opinion post-type alignment rule).
- The policy-check pill shows green.
- If the agent's first iteration produced a section with no stance markers, the guardrail blocked it and the block-log strip shows the one block; the final published post passed.

## J6 — Custom topic with no matching reference file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/references/<custom-topic-slug>.json` exists for the user's topic.

**Steps:**
1. In the App UI, type a custom topic (e.g. `artisanal fermentation techniques`) into the topic field.
2. Select any post type.
3. Click **Run pipeline**.

**Expected:**
- `ResearchTools.searchReferences` returns an empty list.
- The agent's RESEARCH task returns `ResearchNotes` with `references = []` and a `topicSummary` indicating no material was found.
- The workflow advances through OUTLINE, DRAFT, EDIT, PUBLISH on the empty research base.
- The agent's OUTLINE task returns an `Outline` with `sections = []` (the agent's refusal behaviour per `prompts/blog-writer-agent.md`).
- The DRAFT task returns a `Draft` with `title = "(no outline sections)"` and `sections = []`.
- The pipeline completes to `PUBLISHED` with an empty post; nothing crashes; the empty post is honestly empty.
- The policy-check pill shows green (no prohibited content in empty prose).
