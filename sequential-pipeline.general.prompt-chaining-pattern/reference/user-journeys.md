# User journeys — prompt-chaining-workflow

## J1 — Submit a prompt and get a refined document

**Preconditions:** Service running on declared port (`http://localhost:9999/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded prompt `Introduction to vector databases` has matching `src/main/resources/sample-data/templates/` and `citations/` files.

**Steps:**
1. Open `http://localhost:9999/` → App UI tab.
2. From the **Pick a seeded prompt** dropdown, pick `Introduction to vector databases`.
3. Click **Start drafting**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `OUTLINING` within 1 s more.
- Within ~20 s the card reaches `OUTLINED`. The right pane shows the Outline panel with ≥ 2 sections; each row has a non-empty heading and description.
- Within ~30 s more the card reaches `DRAFTED`. The right pane shows ≥ 2 section drafts with body text and a citations table with ≥ 2 entries; every `citationId` in a section draft appears in the citations list.
- Within ~20 s more the card reaches `REFINED`, then `EVALUATED` within 1 s of that. The right pane shows a `RefinedDocument` with title, abstract, and `sections.length == outline.sections.length`; every refined section has at least one citation marker; the quality score chip shows 5/5.
- Total elapsed time: ≤ 90 s on the happy path.

## J2 — Response guardrail blocks a structurally incomplete refined document

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `refine-document.json` includes one entry whose `sections` list contains only 1 section — below the minimum-2-section threshold.

**Steps:**
1. Submit any seeded prompt four times in a row (J1 steps × 4).
2. Watch the fourth submission's lifecycle in the network panel (`/api/drafts/sse`).

**Expected:**
- On the fourth submission's `refineStep`, the agent returns a `RefinedDocument` with 1 section. `OutputGuardrail` rejects it with `quality-violation: sections.size() = 1, minimum is 2`. A `GuardrailRejected{check: "section-count", reason: "..."}` event lands on the entity.
- The rejected response is NEVER committed to `DraftEntity` — there is no `RefinementApplied` event for it.
- The workflow retries `refineStep`. The agent receives the rejection reason appended to its instructions: `RETRY REASON: section-count failed — sections.size() = 1, minimum is 2`. The second iteration produces a compliant document.
- The draft eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected response with its full structured reason.

## J3 — Hallucinated citation flags quality score 1

**Preconditions:** Mock LLM mode. The mock's `refine-document.json` includes one entry whose `bibliography` contains a `Citation` whose `citationId` is absent from the paired `Draft.citations`.

**Steps:**
1. Submit any seeded prompt five times. (The hallucinated-citation entry is selected once in every five runs by the mock's `seedFor(draftId)` modulo.)
2. Watch the live list until a draft's card border highlights red.

**Expected:**
- The flagged draft passes the response guardrail (which checks structural minimums, not citation provenance).
- The eval score chip shows **1** (only the structural-parity check passed) and the rationale reads: *"Citation provenance failed: bibliography entry 'cit-xxxx' does not appear in Draft.citations."*
- The card's border highlights red. The reader knows to inspect this document before publishing.
- The other four drafts in the run scored ≥ 4.

## J4 — Per-step tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded prompt.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines. Filter to entries tagged with the draftId.

**Expected:**
- The OUTLINE task's log entries show only `structurePrompt` and `fetchSectionTemplates` calls.
- The DRAFT task's log entries show only `writeSection` and `gatherCitations` calls.
- The REFINE task's log entries show only `polishSection` and `formatCitation` calls.
- No cross-step calls appear.
- The order of tasks in the log is OUTLINE → DRAFT → REFINE. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Structural parity across the chain

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded prompt `Overview of transformer attention mechanisms` (whose mock-paired `Outline` carries 4 sections).
2. Wait for `EVALUATED`.

**Expected:**
- The recorded `Outline.sections.length` equals 4.
- The recorded `Draft.sectionDrafts.length` equals 4.
- The recorded `RefinedDocument.sections.length` equals 4.
- Every `sectionId` in the outline appears exactly once in the draft and once in the refined document.
- The quality score chip shows 5/5 with rationale "all checks passed". A score of 4 would indicate structural parity failed at one boundary; this journey verifies end-to-end parity held.

## J6 — Custom prompt with no matching template file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/templates/<custom-slug>.json` exists for the user's prompt.

**Steps:**
1. In the App UI, type a custom prompt (e.g. `the geopolitics of deep-sea mining`) into the prompt field.
2. Click **Start drafting**.

**Expected:**
- `OutlineTools.structurePrompt` returns a best-effort list of sections derived from the prompt text alone (no file to load).
- The agent's OUTLINE task returns a non-empty `Outline` (the agent uses its own structural judgment when no template data is available).
- The pipeline advances through DRAFT and REFINE normally.
- The quality score may be lower than 5 if the agent's ad-hoc outline leads to section-count drift downstream, but the pipeline completes without crashing.
- Nothing in the service throws an unhandled exception; the fallback path is the honest result of running without seed data.
