# User journeys — blog-writer-pipeline

## J1 — Submit a topic and get a complete blog post

**Preconditions:** Service running on declared port (`http://localhost:9250/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded topic `The future of developer tooling` has a matching `src/main/resources/sample-data/references/the-future-of-developer-tooling.json` file.

**Steps:**
1. Open `http://localhost:9250/` → App UI tab.
2. From the **Pick a seeded topic** dropdown, pick `The future of developer tooling`.
3. Select style `Technical`.
4. Click **Generate post**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `RESEARCHING` within 1 s more.
- Within ~20 s the card reaches `RESEARCHED`. The right pane shows the Research table with ≥ 4 rows; each row has a non-empty source, url, and keyInsight.
- Within ~20 s more the card reaches `OUTLINED`. The right pane shows ≥ 2 outline sections; every section has a heading and ≥ 2 key points.
- Within ~20 s more the card reaches `DRAFTED`, then `QUALITY_CHECKED` within 1 s. The right pane shows a `BlogPost` with `sections.length == outline.sections.length`, every section body is ≥ 50 words, and the quality score chip shows 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Brand guardrail blocks a short draft

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `draft-post.json` includes one entry whose combined word count is < 300 — this is the deliberately short entry.

**Steps:**
1. Submit any seeded topic three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/posts/sse`).

**Expected:**
- On the third submission's `draftStep`, the agent's first iteration produces a draft under 300 words. `BrandGuardrail` rejects it with `rule: "word-count"` and a detail message naming the actual count.
- A `GuardrailRejected{rule: "word-count", detail: "Draft has N words; minimum is 300."}` event lands on the entity.
- The short draft NEVER becomes a `DraftWritten` event — there is no log line from the entity's event-applier for that candidate.
- The agent's second iteration produces a conformant draft. The card eventually reaches `QUALITY_CHECKED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected draft with its rule and word count.

## J3 — Brand guardrail blocks a forbidden-phrase draft

**Preconditions:** Mock LLM mode. The mock's `draft-post.json` includes one entry whose introduction contains the phrase `paradigm shift`.

**Steps:**
1. Submit any seeded topic. If the mock selects the forbidden-phrase entry (by `seedFor` modulo), the rejection fires on the first DRAFT iteration.

**Expected:**
- `BrandGuardrail` rejects the draft with `rule: "forbidden-phrase"` and a detail excerpt naming the phrase and its position.
- The agent's retry removes the forbidden phrase. The final recorded `DraftWritten` does not contain it.
- The rejection-log strip shows the one rejection with rule `forbidden-phrase`. The accepted draft shows normally in the post detail pane.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider.

**Steps:**
1. Submit any seeded topic.
2. Wait for `QUALITY_CHECKED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the postId.

**Expected:**
- The RESEARCH task's log entries show only `searchTopicReferences` and `fetchKeyPoints` calls.
- The OUTLINE task's log entries show only `generateSectionHeadings` and `assignKeyPoints` calls.
- The DRAFT task's log entries show only `writeSection` and `writeConclusion` calls.
- No cross-phase calls appear. If the mock-LLM short-draft path fired, the guardrail rejection appears as a `guardrail.reject` line before the agent's retry; the rejected response is logged but never written to the entity.
- The order of tasks in the log is RESEARCH → OUTLINE → DRAFT. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Section parity between outline and draft

**Preconditions:** Service running. Any model provider. The seeded topic `How agentic AI changes product management` has a mock-paired `Outline` carrying 3 sections.

**Steps:**
1. Submit `How agentic AI changes product management` with style `Thought leadership`.
2. Wait for `QUALITY_CHECKED`.

**Expected:**
- The recorded `BlogPost.sections.length` equals the recorded `Outline.sections.length` (both 3).
- Every `post.sections[i].sectionId` matches one `outline.sections[i].sectionId`, one-to-one.
- The quality score chip shows 5/5 with rationale "Section coverage, section depth, title fidelity, and section parity all satisfied."

## J6 — Custom topic with no matching reference file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/references/<slug>.json` exists for the user's topic.

**Steps:**
1. In the App UI, type a custom topic (e.g. `artisanal cheese ageing techniques`) into the topic field and click **Generate post**.

**Expected:**
- `ResearchTools.searchTopicReferences` returns an empty list.
- The agent's RESEARCH task returns a `ResearchNotes` with `references = []`.
- The workflow advances to `outlineStep`. The OUTLINE task returns an `Outline` with `proposedTitle = "(no references)"` and `sections = []`.
- The workflow advances to `draftStep`. The DRAFT task returns a `BlogPost` with empty sections.
- The brand guardrail passes (empty sections means word count check is likely < 300 words — the guardrail rejects it; the agent produces the refusal form with an explanatory introduction).
- The quality score chip shows 1 (no sections to score). The rationale names "no outline sections to match."
- The pipeline completes; nothing crashes; the empty or near-empty post is honestly recorded.
