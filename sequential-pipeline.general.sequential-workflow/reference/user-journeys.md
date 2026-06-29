# User journeys — sequential-workflow

## J1 — Submit a job and get a result

**Preconditions:** Service running on declared port (`http://localhost:9509/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded job type `csv-transform` has a matching `src/main/resources/sample-data/job-types/csv-transform.json` file.

**Steps:**
1. Open `http://localhost:9509/` → App UI tab.
2. From the **Job type** dropdown, select `csv-transform`. Enter `quarterly-report-csv` in the **Job name** field.
3. Click **Run workflow**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `VALIDATING` within 1 s more.
- Within ~20 s the card reaches `VALIDATED`. The right pane shows the Validation panel with ≥ 2 field rows; every row has `status = "OK"` and the aggregate `valid = true`.
- Within ~20 s more the card reaches `ENRICHED`. The right pane shows the Enrichment panel with resolved parameters and metadata key/value pairs.
- Within ~20 s more the card reaches `EXECUTED`. The right pane shows the Execution panel with ≥ 2 step rows; each step has a non-empty outcome and at least one artifact chip.
- Within ~20 s more the card reaches `SUMMARIZED`, then `EVALUATED` within 1 s of that. The right pane shows a `JobSummary` with `entries.length == steps.length`, every entry references ≥ 1 artifact id, every artifact id matches an artifact from the Execution panel, and the quality score chip shows 5/5.
- Total elapsed time: ≤ 80 s on the happy path.

## J2 — Phase-gate guardrail blocks a misordered tool call

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `validate-job.json` includes one entry whose `tool_calls` array starts with an EXECUTE-phase tool (`runStep`) — this is the deliberately phase-violating entry.

**Steps:**
1. Submit any seeded job type three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/jobs/sse`).

**Expected:**
- On the third submission's `validateStep`, the agent's first iteration calls `runStep`. `StepGuardrail` rejects it; a `GuardrailRejected{phase: "VALIDATE", tool: "runStep", reason: "phase-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `ExecuteTools.runStep` — there is no log line from the tool body.
- The agent's second iteration falls through to a normal validate sequence (`checkFields` + `verifyConstraints`). The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — Missing artifact reference flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `summarize-job.json` paired entry includes a `SummaryEntry` whose `artifactIds` list contains an id absent from the paired `JobOutput`.

**Steps:**
1. Submit any seeded job type six times. (The missing-artifact entry is selected once in every six runs by the mock's `seedFor(jobId)` modulo.)
2. Watch the live list until a job's card border highlights red.

**Expected:**
- The flagged job lands well-formed (the guardrail only checks phase order, not artifact provenance).
- The quality score chip shows **1** (or 2 depending on which other rules pass) and the rationale reads: *"Artifact provenance failed: entry 's-transform' cites artifactId 'a-XXXX' which does not appear in the recorded JobOutput."*
- The card's border highlights red. The operator knows to inspect this job before using its output downstream.
- The other five jobs in the run scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded job type.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the jobId.

**Expected:**
- The VALIDATE task's log entries show only `checkFields` and `verifyConstraints` calls.
- The ENRICH task's log entries show only `resolveParameters` and `attachContext` calls.
- The EXECUTE task's log entries show only `runStep` and `collectArtifacts` calls.
- The SUMMARIZE task's log entries show only `buildSummaryEntry` and `writeOutcomeStatement` calls.
- No cross-phase calls appear. (If the mock-LLM violation path fires, a single `guardrail.reject` line precedes the agent's retry; the rejected call is logged but never executed.)
- The order of tasks in the log is VALIDATE → ENRICH → EXECUTE → SUMMARIZE. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Entry-step parity

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded job type `report-generation` (whose mock-paired `JobOutput` carries 3 steps).
2. Wait for `EVALUATED`.

**Expected:**
- The recorded `JobSummary.entries.length` equals the recorded `JobOutput.steps.length` (both 3).
- Every `entry.stepId` matches one `step.stepId` from the output, one-to-one.
- If the agent's first iteration produced 2 entries instead of 3 (silent collapse), the on-completion evaluator would have flagged it with an "entry parity" failure and a score of 4. The presence of a score of 5 with rationale "all checks passed" confirms parity held.

## J6 — Job type with no matching catalog file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/job-types/<custom-type>.json` exists for the user's job type.

**Steps:**
1. In the App UI, type a custom job type (e.g., `artisanal-pickle-inventory`) into the Job type field.
2. Click **Run workflow**.

**Expected:**
- `ValidateTools.checkFields` returns field results with `status = "MISSING"` for every declared field.
- The agent's VALIDATE task returns a `ValidationResult` with `valid = false`.
- The workflow advances to `enrichStep`. The ENRICH task returns an `EnrichedJob` with an empty `params` list and minimal metadata.
- The workflow advances to `executeStep`. The EXECUTE task returns a `JobOutput` with `steps = []`.
- The workflow advances to `summarizeStep`. The SUMMARIZE task returns a `JobSummary` with `title = "(no steps executed)"` and `entries = []`.
- The quality score chip shows 1 (step count = 0, entry count = 0 — parity is satisfied, but no other rule scores). The rationale names "no steps to score."
- The pipeline completes; nothing crashes; the empty result is honestly empty.
