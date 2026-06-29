# User journeys — akka-llm-pipeline-workflow

## J1 — Submit a topic and get a finished post

**Preconditions:** Service running on declared port (`http://localhost:9854/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9854/` → App UI tab.
2. From the **Pick a seeded topic** dropdown, pick `The future of edge computing`.
3. Leave the word target at the default (600). Click **Generate post**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `OUTLINING` within 1 s more.
- Within ~15 s the card reaches `OUTLINED`. The right pane shows the Outline sections table with ≥ 2 rows; each row has a non-empty heading and at least 2 key-point bullets.
- Within ~25 s more the card reaches `DRAFTED`, then `REVIEWING` within 1 s.
- Within ~2 s the card reaches `READY`. The right pane shows the full draft — title, introduction, section bodies, conclusion — and the editorial-review chip reads `PASSED`. Word count is displayed and is within ±20% of 600.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Schema guardrail blocks a malformed outline

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `outline-activity.json` includes one entry with `sections = []` (the deliberately invalid entry selected on every 4th post).

**Steps:**
1. Submit any seeded topic four times in a row.
2. Watch the fourth submission's lifecycle in the App UI live list.

**Expected:**
- On the fourth submission's `outlineStep`, the mock returns an outline with empty sections. `SchemaGuardrail` catches it at the `after-llm-response` hook.
- A `ValidationFailed{field: "sections", reason: "sections is empty; at least one OutlineSection is required"}` event lands on the entity.
- The post transitions to `FAILED`. The card shows the red `FAILED` status pill.
- The right pane shows the validation-failure strip with the field name, reason, and timestamp. Phase panels 1 and 2 are not rendered.
- The draft activity is never called — there is no `DraftStarted` event in the entity log.

## J3 — Editorial guardrail flags a prohibited-topic draft

**Preconditions:** Service running with the mock LLM selected. The mock's `draft-activity.json` includes one entry whose introduction contains a prohibited phrase from `prohibited-phrases.txt`.

**Steps:**
1. Submit any seeded topic enough times for the flagged mock entry to be selected (varies by seed; typically within 6 runs).
2. Watch the live list until a card shows the amber `FLAGGED` status pill.

**Expected:**
- The flagged post's outline passes `SchemaGuardrail` normally (the outline itself is valid).
- After the draft activity returns, `EditorialGuardrail` catches the prohibited phrase in the `before-agent-response` hook.
- `PostFlagged{reason: "editorial-flag: prohibited-topic: phrase '...' found in introduction"}` lands on the entity.
- The card's border highlights amber. The editorial-review chip reads `FLAGGED` with the structured reason.
- The draft text is visible in the right pane (the draft exists; it is held, not discarded).
- The other posts in the run that did not hit the prohibited-phrase entry show `READY` with review chip `PASSED`.

## J4 — Draft receives only the outline (no raw topic)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit the seeded topic `Zero-trust security in practice`.
2. Wait for `READY`.
3. Inspect the service log for the draft activity's LLM call context. Filter to lines tagged with the postId.

**Expected:**
- The `outlineStep` log shows the topic string `"Zero-trust security in practice"` in the LLM call's input.
- The `draftStep` log shows only the serialised `PostOutline` in the LLM call's input — no occurrence of the raw topic string `"Zero-trust security in practice"`.
- The order in the log is `outlineStep` start → `outlineStep` complete → `draftStep` start → `draftStep` complete. No cross-step context leakage appears.

## J5 — Section coverage mismatch is caught by editorial guardrail

**Preconditions:** Service running with the mock LLM. A mock draft entry whose `DraftSection` headings do not fully cover the paired outline's `OutlineSection` headings (one heading missing or renamed).

**Steps:**
1. Submit the seeded topic `Building resilient distributed systems` with word target 800.
2. Wait for the `FLAGGED` or `READY` result (depends on which mock entry is selected).

**Expected (flagged path):**
- `EditorialGuardrail` applies the section-coverage rule: every `OutlineSection.heading` must have a matching `DraftSection.heading` (case-insensitive).
- The rule fails for the missing heading; `PostFlagged{reason: "editorial-flag: section-coverage: outline heading 'Failure modes' has no matching draft section"}` lands on the entity.
- The post is `FLAGGED`. The review chip shows the specific unmatched heading.

**Expected (passing path):**
- If the mock selects a well-formed entry, the coverage check passes and the post reaches `READY`.

## J6 — Custom topic with short word target

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Type a custom topic: `Introduction to content delivery networks`.
2. Set word target to 300.
3. Click **Generate post**.

**Expected:**
- `OutlineActivity` produces an outline with 2 sections (appropriate for a 300-word target).
- `SchemaGuardrail` accepts the outline.
- `DraftActivity` produces a draft with 2 sections, word count within ±20% of 300 (240–360 words).
- `EditorialGuardrail` applies the word-count rule: if the draft is within range, `READY`; if out of range, `FLAGGED` with reason `"editorial-flag: word-count: 410 words is more than 20% above target 300"`.
- No crashes; partial states are visible in the UI at each transition.
