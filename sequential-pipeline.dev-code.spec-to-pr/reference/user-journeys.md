# User journeys — spec-to-pr

## J1 — Submit a spec and reach AWAITING_REVIEW

**Preconditions:** Service running on declared port (`http://localhost:9745/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded spec `Add rate-limiting middleware` has a matching `src/main/resources/sample-data/codebase/rate-limiting.json` file.

**Steps:**
1. Open `http://localhost:9745/` → App UI tab.
2. From the **Pick a seeded spec** dropdown, pick `Add rate-limiting middleware`.
3. Click **Run pipeline**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `PARSING` within 1 s more.
- Within ~20 s the card reaches `PARSED`. The right pane shows the Parsed-spec requirements table with ≥ 1 row; each row has a non-empty reqId, text, priority, and at least one affectedFile.
- Within ~20 s more the card reaches `PLANNED`. The right pane shows ≥ 1 file change; each row has a filePath present in the mock manifest.
- Within ~20 s more the card reaches `AWAITING_REVIEW`. The right pane shows the Draft PR (title, description, per-file diff blocks) and the reviewer action bar with Approve and Reject buttons enabled.
- Total elapsed time to `AWAITING_REVIEW`: ≤ 60 s on the happy path.

## J2 — Phase-gate guardrail blocks a misordered tool call

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `parse-spec.json` includes one entry whose `tool_calls` array starts with a DRAFT-phase tool (`composePrTitle`) — the deliberately phase-violating entry.

**Steps:**
1. Submit any seeded spec three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/runs/sse`).

**Expected:**
- On the third submission's `parseStep`, the agent's first iteration calls `composePrTitle`. `WriteGuardrail` rejects it; a `GuardrailRejected{phase: "PARSE", tool: "composePrTitle", reason: "phase-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `DraftTools.composePrTitle` — there is no log line from the tool body.
- The agent's second iteration falls through to a normal parse sequence (`extractRequirements` + `identifyAffectedFiles`). The run eventually reaches `AWAITING_REVIEW` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — Reviewer rejects the draft PR

**Preconditions:** Service running. Any model provider. A run has reached `AWAITING_REVIEW` (J1 completed).

**Steps:**
1. Complete J1 so a run is in `AWAITING_REVIEW`.
2. In the reviewer action bar, enter reviewer ID `eng-alice` and comment `Diffs out of scope — please narrow to the middleware layer only.`
3. Click **Reject**.

**Expected:**
- The run immediately transitions to `REJECTED`.
- The card's status pill turns red.
- The reviewer action bar disappears and is replaced by the rejection decision badge showing reviewer ID, comment, and timestamp.
- No CI run is triggered — `CiScorer` is never called.
- `GET /api/runs/{id}` returns `status: "REJECTED"` and `reviewDecision.decision: "rejected"` in the response. `ciResult` is `null`.

## J4 — CI gate blocks a PR with a failing test check

**Preconditions:** Service running with the mock LLM selected. The mock's `plan-changes.json` includes one entry that proposes a `changeType = "delete"` on a file with a `Test` counterpart in the manifest — this triggers the CI test check to fail.

**Steps:**
1. Submit seeded specs until a run whose `ChangePlan` contains the delete-test entry reaches `AWAITING_REVIEW`. (The failing entry is selected by `seedFor(runId)` modulo.)
2. In the reviewer action bar, enter reviewer ID `eng-bob` and click **Approve**.

**Expected:**
- The run transitions to `CI_RUNNING` immediately after approval.
- Within ~2 s, the run transitions to `CI_FAILED`.
- The CI panel in the right pane shows a 3-row check table: `compile` PASS, `test` FAIL (message: "Deleting src/... which has a test counterpart src/...Test.java"), `lint` PASS.
- The card's status pill turns red with `CI_FAILED`.
- The run does NOT reach `MERGE_READY`. The reviewer action bar does not reappear — `CI_FAILED` is terminal.

## J5 — Full happy path to MERGE_READY

**Preconditions:** Service running. Any model provider. A run has passed J1 and a reviewer approves it (mock CI will pass).

**Steps:**
1. Complete J1 so a run is in `AWAITING_REVIEW`.
2. Confirm the change plan in the right pane: all filePaths are in the manifest, no deletes on test-paired files, all diffs ≤ 200 lines.
3. In the reviewer action bar, enter reviewer ID `eng-alice` and click **Approve**.

**Expected:**
- The run transitions to `CI_RUNNING` within 1 s of approval.
- Within ~3 s, the run transitions to `CI_PASSED` then immediately to `MERGE_READY`.
- The CI panel shows all three checks passing with green badges.
- The card's status pill turns green with `MERGE_READY`.
- `GET /api/runs/{id}` returns `status: "MERGE_READY"`, `ciResult.passed: true`, and all three `ciResult.checks[i].passed: true`.
