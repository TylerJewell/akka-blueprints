# User journeys — deterministic-multi-stage-agent-pipeline

## J1 — Submit a prompt and get a story

**Preconditions:** Service running on declared port (`http://localhost:9724/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded prompt `The lighthouse keeper's last night` has a matching `src/main/resources/sample-data/genres/the-lighthouse-keepers-last-night.json` file.

**Steps:**
1. Open `http://localhost:9724/` → App UI tab.
2. From the **Pick a seeded prompt** dropdown, pick `The lighthouse keeper's last night`.
3. Click **Run pipeline**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `OUTLINING` within 1 s more.
- Within ~20 s the card reaches `OUTLINED`. The right pane shows the Outline panel with ≥ 2 beats; each beat has a non-empty `beatId`, `text`, and `position`. The genre chip shows a non-empty string.
- Within ~20 s more the card reaches `BODY_WRITTEN`. The right pane shows ≥ 2 paragraphs; every `paragraph.beatId` matches a `beat.beatId` from the outline.
- Within ~20 s more the card reaches `ENDING_WRITTEN`, then `VALIDATED` within 1 s of that. The right pane shows a non-empty closing text, ≥ 1 arc, and the validation score chip shows 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Stage-output guardrail blocks a structurally incomplete outline

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `outline-story.json` includes one entry whose `Outline.beats` list contains exactly one Beat — this is the deliberately incomplete entry.

**Steps:**
1. Submit any seeded prompt three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/stories/sse`).

**Expected:**
- On the third submission's `outlineStep`, the agent's first iteration returns an `Outline` with one Beat. `StageOutputGuardrail` rejects it with reason `"structural-violation: beats.size() must be >= 2, saw 1"`; a `GuardrailRejected{stage: "OUTLINE", field: "beats.size", ...}` event lands on the entity.
- The incomplete Outline NEVER reaches `StoryPipelineWorkflow`'s advancement step — the workflow does not write `OutlineProduced` until the guardrail accepts.
- The agent's second iteration returns a valid Outline (≥ 2 beats). The card eventually reaches `VALIDATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected result with its full structured reason.

## J3 — Stage-output guardrail blocks an orphaned paragraph

**Preconditions:** Mock LLM mode. The mock's `write-body.json` includes one entry where a `Paragraph.beatId` does not match any beat in the paired Outline.

**Steps:**
1. Submit any seeded prompt enough times for the orphaned-paragraph mock entry to be selected (the mock's `seedFor(storyId)` modulo schedule determines when).
2. Watch the live list for a card that transitions through `OUTLINING` successfully, then pauses at `BODY_WRITING` before resuming.

**Expected:**
- The `bodyStep`'s first iteration returns a `Body` with one orphaned paragraph. `StageOutputGuardrail` rejects it with reason `"structural-violation: Paragraph.beatId 'beat-xx' not found in outline beats"`.
- A `GuardrailRejected{stage: "WRITE_BODY", field: "Paragraph.beatId", ...}` event lands on the entity.
- The agent's next iteration returns a corrected Body with all paragraphs referencing valid beat ids. The card eventually reaches `VALIDATED`.
- The rejection-log strip on the right pane shows the one rejected body with the orphaned beatId named in the reason.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded prompt.
2. Wait for `VALIDATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the storyId.

**Expected:**
- The OUTLINE task's log entries show only `classifyGenre` and `generateBeats` calls.
- The WRITE_BODY task's log entries show only `expandBeat` and `linkParagraphs` calls.
- The WRITE_ENDING task's log entries show only `resolveArcs` and `composeClosure` calls.
- No cross-stage calls appear.
- The order of tasks in the log is OUTLINE → WRITE_BODY → WRITE_ENDING. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Beat coverage validation

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded prompt `A city that forgot the sun` (whose mock-paired `Outline` carries 4 beats).
2. Wait for `VALIDATED`.

**Expected:**
- The recorded `Body.paragraphs` contains ≥ 4 entries, with each of the 4 beat ids from the Outline referenced by at least one paragraph.
- The `ValidationResult.score` is 5 and `rationale` reads `"Genre detection, beat coverage, arc linkage, and title consistency all satisfied."`.
- If any beat had no covering paragraph, the evaluator would flag it with a "beat coverage" failure and reduce the score by 1. The presence of a full score confirms coverage holds.

## J6 — Custom prompt with no matching genre file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/genres/<prompt-slug>.json` exists for the user's prompt.

**Steps:**
1. In the App UI, type a custom prompt (e.g. `A detective who only works at noon`) into the prompt field.
2. Click **Run pipeline**.

**Expected:**
- `OutlineTools.generateBeats` returns an empty list.
- The agent's OUTLINE task returns an `Outline` with `genre = ""` and `beats = []`.
- `StageOutputGuardrail` rejects this because `beats.size() < 2`; a `GuardrailRejected` event lands.
- After exhausting its retry budget without producing a valid outline, the workflow fails over to the error step. The story lands in `FAILED` with the partial state preserved.
- The UI shows the `FAILED` status pill and the rejection-log strip with each failed attempt logged. Nothing crashes; the empty trajectory is honestly recorded.
