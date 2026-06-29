# User journeys — parallel-hiring-reviewers

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on declared port (9853); valid model-provider API key set (or mock LLM selected).

**Steps:**
1. Open `http://localhost:9853/`. App UI tab is visible.
2. In the Candidate name field type "Jordan Smith"; in the Role applied for field type "Senior Software Engineer"; in the Résumé text area paste two clean paragraphs describing technical experience and prior projects. Click Submit.
3. A new evaluation card appears with status INTAKE.

**Expected:**
- Within 1 s, status transitions to EVALUATING via SSE.
- Within 60 s, status transitions to DECIDED.
- The expanded view shows three perspective-review blocks (HR, Manager, Team), each with a decision (`ADVANCE`/`HOLD`/`DECLINE`), a 1–5 score, and 1–5 findings; and one overall recommendation block with a decision (`HIRE`/`FURTHER_REVIEW`/`REJECT`), a 60–120 word rationale, and `guardrailVerdict: "ok"`. No failure reason.

## J2 — Protected-attribute redaction at intake

**Preconditions:** As J1.

**Steps:**
1. Submit a profile whose résumé text contains the phrases "married father of two" and "age 42, originally from".

**Expected:**
- `GET /api/evaluations/{id}` returns a `redactedResumeText` in which the marital-status and age phrases are replaced with `[MARITAL_STATUS]` and `[AGE]` (and the national-origin phrase with `[ORIGIN]`); the raw phrases appear nowhere in the response.
- `redactionCount` is at least 2.
- The evaluation card shows a redaction-count chip.
- The persisted evaluation never contains the raw résumé text — confirmable by inspecting the entity via the backoffice.

## J3 — Degraded reviewer

**Preconditions:** As J1, plus one reviewer's step timeout set to 1 s (configurable via env var `TEAM_TIMEOUT_MS=1000` or by editing `application.conf`).

**Steps:**
1. Submit any candidate profile.

**Expected:**
- Evaluation progresses INTAKE → EVALUATING → DEGRADED.
- The expanded view shows the two reviewer perspectives that returned populated, the timed-out perspective empty, the overall rationale acknowledging the missing perspective, and `failureReason: "reviewer timeout: TeamReviewer"`.
- The overall recommendation is aggregated from the available perspectives only.

## J4 — Guardrail block

**Preconditions:** As J1, plus an override that forces the coordinator's aggregated recommendation to fail the structural vetter (e.g., submit a profile with the role "test-policy-block", which the deterministic vetter is configured to reject — see `application.conf` test-mode override).

**Steps:**
1. Submit a profile with the role applied for set to "test-policy-block".

**Expected:**
- Evaluation progresses INTAKE → EVALUATING → BLOCKED.
- The expanded view shows the three perspective reviews still populated (visible for audit) but the overall recommendation block carries `guardrailVerdict: "blocked: <reason>"` and the evaluation is in BLOCKED status.

## J5 — Fairness eval scoring

**Preconditions:** At least one DECIDED evaluation exists with no `fairnessScore`. The `FairnessSampler` schedule reduced to 30 s for the test (`FAIRNESS_SAMPLER_SECONDS=30`).

**Steps:**
1. Submit any candidate profile; wait for DECIDED.
2. Wait 30 s.

**Expected:**
- The evaluation card shows a fairness-score chip (1–5) and the rationale is visible in the expanded view.
- The evaluation's row in `/api/evaluations` includes `fairnessScore` and `fairnessRationale` populated.
- The evaluation's status is unchanged (still DECIDED); only the score fields are added.

## J6 — Background load from the simulator

**Preconditions:** Service running; no UI interaction.

**Steps:**
1. Open the App UI tab and wait.

**Expected:**
- Without any submission, a new evaluation appears roughly every 90 s, seeded from `sample-events/candidate-profiles.jsonl`, and runs the full INTAKE → EVALUATING → DECIDED path on its own. At least one seeded profile carries protected-attribute markers and shows a non-zero redaction count.
