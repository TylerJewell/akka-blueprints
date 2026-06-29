# User journeys — peerreview

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on declared port (9607); valid model-provider API key set (or mock LLM selected).

**Steps:**
1. Open `http://localhost:9607/`. App UI tab is visible.
2. In the Title field type "Q3 launch post"; in the Document body field paste two clean paragraphs about a product launch. Click Submit.
3. A new review card appears with status INTAKE.

**Expected:**
- Within 1 s, status transitions to REVIEWING via SSE.
- Within 60 s, status transitions to SYNTHESISED.
- The expanded view shows three axis-review blocks (Technical, Style, Compliance), each with a verdict (`PASS`/`REVISE`/`FAIL`), a 1–5 score, and 1–5 findings; and one overall verdict block with a verdict (`APPROVE`/`REVISE`/`REJECT`), a 60–120 word summary, and `guardrailVerdict: "ok"`. No failure reason.

## J2 — PII redaction at intake

**Preconditions:** As J1.

**Steps:**
1. Submit a document whose body contains the line "Reach Jane Doe at jane.doe@example.com or 555-123-4567 for comment."

**Expected:**
- `GET /api/reviews/{id}` returns a `redactedBody` in which the email and phone number are replaced with `[EMAIL]` and `[PHONE]` (and the name with `[NAME]`); the raw email and phone strings appear nowhere in the response.
- `redactionCount` is at least 2.
- The review card shows a redaction-count chip.
- The persisted review never contains the raw body — confirmable by inspecting the entity via the backoffice.

## J3 — Degraded reviewer

**Preconditions:** As J1, plus one reviewer's step timeout set to 1 s (configurable via env var `TECHNICAL_TIMEOUT_MS=1000` or by editing `application.conf`).

**Steps:**
1. Submit any document.

**Expected:**
- Review progresses INTAKE → REVIEWING → DEGRADED.
- The expanded view shows the two reviewers that returned populated, the timed-out axis empty, the overall summary acknowledging the missing axis, and `failureReason: "reviewer timeout: TechnicalReviewer"`.
- The overall verdict is reconciled from the available axes only.

## J4 — Guardrail block

**Preconditions:** As J1, plus an override that forces the moderator's reconciled verdict to fail the structural vetter (e.g., submit the title "test-policy-block", which the deterministic vetter is configured to reject — see `application.conf` test-mode override).

**Steps:**
1. Submit the document titled "test-policy-block".

**Expected:**
- Review progresses INTAKE → REVIEWING → BLOCKED.
- The expanded view shows the three axis reviews still populated (visible for audit) but the overall verdict block carries `guardrailVerdict: "blocked: <reason>"` and the review is in BLOCKED status.

## J5 — Consistency eval scoring

**Preconditions:** At least one SYNTHESISED review exists with no `consistencyScore`. The `EvalSampler` schedule reduced to 30 s for the test (`EVAL_SAMPLER_SECONDS=30`).

**Steps:**
1. Submit any document; wait for SYNTHESISED.
2. Wait 30 s.

**Expected:**
- The review card shows a consistency-score chip (1–5) and the rationale is visible in the expanded view.
- The review's row in `/api/reviews` includes `consistencyScore` and `consistencyRationale` populated.
- The review's status is unchanged (still SYNTHESISED); only the score fields are added.

## J6 — Background load from the simulator

**Preconditions:** Service running; no UI interaction.

**Steps:**
1. Open the App UI tab and wait.

**Expected:**
- Without any submission, a new review appears roughly every 60 s, seeded from `sample-events/review-submissions.jsonl`, and runs the full INTAKE → REVIEWING → SYNTHESISED path on its own. At least one seeded document carries PII and shows a non-zero redaction count.
