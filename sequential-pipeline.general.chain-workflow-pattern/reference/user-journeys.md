# User journeys — chain-workflow

## J1 — Submit content and get a document

**Preconditions:** Service running on declared port (`http://localhost:9974/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded input `Product release notes draft` has a matching `src/main/resources/sample-data/inputs/product-release-notes-draft.json` file.

**Steps:**
1. Open `http://localhost:9974/` → App UI tab.
2. From the **Pick a seeded input** dropdown, pick `Product release notes draft`.
3. Click **Run chain**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `EXTRACTING` within 1 s more.
- Within ~20 s the card reaches `EXTRACTED`. The right pane shows the Extracted-facts table with ≥ 3 rows; each row has a non-empty factId, text, category, and sourceSpan.
- Within ~20 s more the card reaches `REFINED`. The right pane shows ≥ 3 refined facts; every `refinedFact.sourceFactId` matches a `fact.factId` from the extracted table.
- Within ~20 s more the card reaches `FORMATTED`, then `EVALUATED` within 1 s of that. The right pane shows a `Document` with at least 2 sections, every section has at least one citation, every citation's `refinedId` matches a refined fact, and the quality score chip shows 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Output guardrail blocks an empty extract result

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `extract-structure.json` includes one entry whose returned `ExtractedStructure` has an empty `facts` list.

**Steps:**
1. Submit any seeded input three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/documents/sse`).

**Expected:**
- On the third submission's `extractStep`, the agent's first iteration returns a `ExtractedStructure` with `facts = []`. `OutputGuardrail` rejects it; a `GuardrailRejected{stage: "EXTRACT", failedField: "facts", reason: "output-validation-failure: facts list is empty..."}` event lands on the entity.
- The invalid result is NEVER written to the entity — there is no `StructureExtracted` event in the log for that iteration.
- The agent's second iteration returns a valid `ExtractedStructure` with ≥ 3 facts. The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected output with its full structured reason.

## J3 — Uncited section flags quality score 1

**Preconditions:** Mock LLM mode. The mock's `format-document.json` includes one entry whose first `Section` has an empty `citations` list.

**Steps:**
1. Submit any seeded input six times. (The uncited-section entry is selected once in every six runs by the mock's `seedFor(documentId)` modulo.)
2. Watch the live list until a document's card border highlights red.

**Expected:**
- The flagged document lands well-formed (the output guardrail validates per-field shape at the stage level, but cross-stage citation completeness is the evaluator's check).
- The quality score chip shows **1** (or 2 depending on which other rules pass) and the rationale reads: *"Citation completeness failed: section 'X' has no citations."*
- The card's border highlights red. The reader knows to inspect this document before acting on it.
- The other five documents in the run scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded input.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the documentId.

**Expected:**
- The EXTRACT task's log entries show only `parseInput` and `classifyFacts` calls.
- The REFINE task's log entries show only `polishFact` and `mergeDuplicates` calls.
- The FORMAT task's log entries show only `buildSection` and `composeSummary` calls.
- No cross-stage calls appear.
- The order of tasks in the log is EXTRACT → REFINE → FORMAT. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Orphaned-fact detection via quality eval

**Preconditions:** Service running. Mock LLM mode. One mock entry for REFINE produces a `RefinedFact` whose `sourceFactId` does not match any `Fact.factId` in the upstream `ExtractedStructure`.

**Steps:**
1. Submit the seeded input `Technical specification excerpt`.
2. Wait for `EVALUATED`.
3. Check the quality score chip.

**Expected:**
- The output guardrail accepts the REFINE result (it checks referential integrity against the recorded entity state, so a mismatched `sourceFactId` triggers a rejection at the guardrail level and the agent retries). On a clean retry, the document reaches `EVALUATED` with a valid `RefinedContent`.
- If the mock forces the orphaned fact past the guardrail (via a guardrail bypass path in the test fixture), the quality scorer's fact-coverage check fires: the orphaned `Fact.factId` is unreferenced, producing a score of 4 with rationale naming the uncovered fact.
- Either path confirms the governance chain: the guardrail catches referential violations at stage time; the evaluator catches coverage gaps at document time.

## J6 — Empty input produces an honest empty document

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the App UI, submit a raw input consisting of only whitespace (e.g. a single space or empty string).
2. Click **Run chain**.

**Expected:**
- `ExtractTools.parseInput` returns an empty list.
- The agent's EXTRACT task returns an `ExtractedStructure` with `facts = []`.
- The output guardrail rejects the empty-facts result; the agent retries. If the mock LLM also returns empty (no seeded input key matched), the workflow exhausts retries and transitions to `FAILED` after `evalStep` is skipped. The UI shows the card as `FAILED` with a partial state.
- Alternatively, if the agent is permitted to return an empty `ExtractedStructure` explicitly (a legitimate edge case), the REFINE task returns `RefinedContent{refinedFacts: []}`, the FORMAT task returns `Document{title: "(no content to format)", sections: []}`, and the quality score chip shows 1 with rationale "no facts to score." Nothing crashes; the pipeline completes honestly.
