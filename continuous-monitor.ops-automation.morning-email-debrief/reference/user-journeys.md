# User journeys — morning-email-debrief

## J1 — Email batch arrives and debrief assembles

**Preconditions:** Service running on port 9855; valid model-provider API key set or mock LLM configured; `MailboxPoller` enabled.

**Steps:**
1. Open `http://localhost:9855/` → App UI tab.
2. Wait up to 60 s for the first simulated email batch.

**Expected:**
- Each email in the batch appears in the email list with status RECEIVED, then within 1 s transitions to SANITIZED (raw subjects are never displayed).
- Within a further 30 s each email transitions to SUMMARISED with a `DebriefEntry` priority chip visible.
- The debrief run status advances from CREATED → ASSEMBLING → READY.
- The `narrativeSummary` appears in the right panel; the per-entry table lists all emails.

## J2 — No raw PII in the assembled digest

**Preconditions:** Service running with debug logging enabled (`LOG_LEVEL=DEBUG`). A batch includes at least one email with a real email address in the body.

**Steps:**
1. Allow the batch to process fully (debrief reaches READY).
2. Inspect the service log for the `EmailSummarizerAgent` call payload.
3. Inspect the service log for the `DebriefAssemblerAgent` call payload.

**Expected:**
- Both LLM call payloads contain `[REDACTED-EMAIL]` or a similar redacted token — never the raw address.
- The assembled `narrativeSummary` contains no raw identifiers.
- `EmailEntity.incoming` (the audit log) still holds the raw value; `EmailEntity.sanitized` holds the redacted form.

## J3 — Eval score appears on a completed debrief

**Preconditions:** At least one READY debrief with no `evalScore`. `EvalRunner` schedule reduced for the test (`EVAL_RUNNER_SECONDS=60`).

**Steps:**
1. Wait for a debrief to reach READY.
2. Wait 60 s.

**Expected:**
- The debrief run card shows an eval score chip (1–5).
- The detail panel's eval section shows the score and one-sentence rationale.
- `GET /api/debrief/{runId}` includes `evalScore` and `evalRationale` populated.

## J4 — SSE stream delivers live status updates

**Preconditions:** Service running. A batch is in progress.

**Steps:**
1. Open `http://localhost:9855/` → App UI tab.
2. Observe the left panel while a new batch arrives.

**Expected:**
- Email cards appear and update status (RECEIVED → SANITIZED → SUMMARISED) without a page reload.
- The debrief run card updates from CREATED → ASSEMBLING → READY in real time.
- No manual refresh is needed at any point; the UI receives `debrief-update` and `email-update` SSE events.

## J5 — Failed assembly does not lose the email records

**Preconditions:** Service running with the assembler's step timeout artificially shortened (inject a delay in the mock assembler).

**Steps:**
1. Trigger a batch.
2. Allow `assembleStep` to time out.

**Expected:**
- The debrief run transitions to FAILED; `DebriefFailed` is emitted.
- Individual `EmailEntity` records remain in SUMMARISED status — email-level data is not lost.
- The FAILED debrief appears in the run list with a red badge.
- No exception propagates to the UI; the SSE stream continues.
