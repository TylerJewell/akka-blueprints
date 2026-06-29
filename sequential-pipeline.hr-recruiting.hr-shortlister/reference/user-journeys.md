# User journeys — hr-shortlister

## J1 — Submit a resume and reach the approval queue

**Preconditions:** Service running on declared port (`http://localhost:9979/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded profile `Software Engineer L3 candidate` has a matching `src/main/resources/sample-data/profiles/swe-l3.json` file. The seeded job `swe-l3-2026` has a matching `src/main/resources/sample-data/jobs/swe-l3-2026.json` criteria config.

**Steps:**
1. Open `http://localhost:9979/` → App UI tab.
2. From the **Pick a seeded profile** dropdown, pick `Software Engineer L3 candidate`. Select job `swe-l3-2026`.
3. Click **Submit application**.

**Expected:**
- The new card appears in the live list with status `RECEIVED` within 1 s, then `PARSING` within 1 s more.
- Within ~20 s the card reaches `SCORED`. The right pane's profile panel shows extracted skills (≥ 3), experience years, and education. The score panel shows ≥ 3 criterion rows, each with a score and justification. Overall score bar is visible.
- Within ~20 s more the card reaches `AWAITING_APPROVAL`. The agent decision panel shows a `SHORTLIST` / `HOLD` / `REJECT` badge with rationale and confidence score. A yellow approval banner is visible.
- Total elapsed time to `AWAITING_APPROVAL`: ≤ 60 s.
- The recruiter clicks **Approve**. Within ~5 s the card reaches `WRITTEN`. The ERPNext record id (`APP-a-...`) appears in the detail pane.

## J2 — Special-category sanitizer intercepts a protected attribute

**Preconditions:** Service running. The seeded profile `Data Analyst entry-level candidate` (`src/main/resources/sample-data/profiles/analyst-entry.json`) contains an explicit `"age": "26"` field.

**Steps:**
1. Pick the `Data Analyst entry-level candidate` profile, select job `analyst-entry-2026`.
2. Click **Submit application**.
3. Wait for `SCORED`.

**Expected:**
- Before `evaluateCriteria` executes during `scoreStep`, `SpecialCategoryGuardrail` replaces `age` with `"[REDACTED]"` in the `Profile` object passed to the tool.
- A `SpecialCategoryRedacted{fieldName: "age", ...}` event lands on the entity.
- The parsed profile panel shows `age: [REDACTED]` with an orange chip (the field was present but is now redacted). No other field in the profile panel shows the original age value.
- The criterion justifications in the score panel do not reference age.
- The card has a small orange dot in the live list.
- The application proceeds to `AWAITING_APPROVAL` and completes normally after recruiter approval.

## J3 — Recruiter overrides a REJECT decision

**Preconditions:** Service running with the mock LLM. The `Product Manager mid-level candidate` mock entry includes a `ShortlistDecision` with `decision = "REJECT"`.

**Steps:**
1. Submit the `Product Manager mid-level candidate` profile for job `pm-mid-2026`.
2. Wait for `AWAITING_APPROVAL`.
3. In the approval panel, click **Override**. Select `SHORTLIST` from the Decision dropdown. Enter `"Strong product sense evident in project descriptions despite below-threshold criteria score."` as the override rationale.
4. Click **Confirm override**.

**Expected:**
- A `RecruiterOverridden{decision: "SHORTLIST", overrideRationale: "Strong product sense..."}` event lands on the entity.
- The card advances to `APPROVED` then `WRITTEN`.
- The recruiter decision section shows an `Overridden` badge, the recruiter id, the override decision, and the override rationale.
- The ERPNext adapter stub is called with the overriding `SHORTLIST` decision, not the agent's `REJECT`.
- The agent decision panel still shows the original `REJECT` badge so the decision history is preserved.

## J4 — Fairness drift detected after a biased batch

**Preconditions:** Service running with the mock LLM. The seeded mock-response data includes a batch of 30+ applications where the gender cohort `"M"` has a shortlisting rate of ~0.65 and cohort `"F"` has a rate of ~0.45 (ratio 1.44 > threshold 1.20).

**Steps:**
1. Submit the 30 seeded applications by running the bulk-submit script in `src/main/resources/sample-data/bulk-submit.sh` (or submit manually one by one).
2. Approve all applications to move them to `WRITTEN` (required for the fairness scanner to count outcomes).
3. Trigger `FairnessDriftWorkflow` early by calling `POST /api/internal/trigger-drift-scan` (dev-only endpoint).
4. Open the Eval Matrix tab.

**Expected:**
- The Fairness Drift panel shows a `DriftReport` for dimension `gender` with `cohortA: "M"`, `cohortB: "F"`, `ratio: ~1.44`, `breached: true`.
- The panel renders with a red border and the ratio prominently displayed.
- The `nationality` dimension may show `breached: false` if the cohort rates are within threshold.
- No application in the batch was blocked or modified by the drift detector — the monitor is non-blocking.

## J5 — Phase isolation verified via audit log

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded profile.
2. Wait for `AWAITING_APPROVAL`. Then approve.
3. Inspect the service log for tool-call lines. Filter to entries tagged with the applicationId.

**Expected:**
- The PARSE task's log entries show only `extractProfile` and `normalizeEducation` calls.
- The SCORE task's log entries show only `evaluateCriteria` and `computeOverallScore` calls. Each is preceded by a `sanitizer.redact` log line (if any protected fields were present) or a `sanitizer.pass` line (if none were).
- The DECIDE task's log entries show only `classifyDecision` and `generateRationale` calls.
- No cross-phase calls appear. The order in the log is PARSE → SCORE → DECIDE.
- `erpNextStep` log shows the stub response with a synthetic ERPNext id only after the recruiter decision log line.

## J6 — Empty resume produces a well-formed empty application

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Clear the resume textarea and submit with just a job id selected.

**Expected:**
- `ParseTools.extractProfile` returns a `Profile` with `applicantName = "(empty resume)"` and all other fields null.
- The agent's PARSE task returns that minimal profile.
- The SCORE task returns a `CandidateScore` with all criterion scores at 0 and `overallScore = 0`.
- The DECIDE task returns `ShortlistDecision{decision: "REJECT", rationale: "No scorable content in the submitted profile.", confidenceScore: 0}`.
- The application reaches `AWAITING_APPROVAL` with the `REJECT` recommendation visible. The recruiter can override.
- Nothing crashes. The empty application is honestly empty.
