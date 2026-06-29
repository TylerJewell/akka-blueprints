# User journeys — akka-router-pattern

## J1 — Content request: end-to-end handoff to ContentSpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `TaskSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated request seeded as content-flavoured to drop.

**Expected:**
- The request appears with status `RECEIVED` and within ~1 s transitions to `CLASSIFIED`. The domain chip shows `CONTENT`.
- Within ~5 s the status pill shows `GUARDRAIL_PASSED`; the guardrail block shows a green "allowed" check.
- The status pill moves to `ROUTED_CONTENT` then `EXECUTING`.
- The right column populates with the `ContentSpecialist` result: a result title, a complete written output, status `COMPLETED`, and a `content` specialist tag.
- Within ~10 s of the classification decision, the routing score chip shows a number 1–5 with a rationale.
- `finishedAt` is set; status is `COMPLETED`.

## J2 — Code request: end-to-end handoff to CodeSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a code-flavoured request.

**Expected:**
- Classification emits `domain = CODE`. Status moves to `ROUTED_CODE`.
- `CodeSpecialist` returns a `TaskResult` with a code-focused result body (fenced code block present).
- Guardrail allows. Status `COMPLETED`.
- `ContentSpecialist` and `DataSpecialist` are never invoked for this request.

## J3 — Data request: end-to-end handoff to DataSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Wait for a data-flavoured request to drop.

**Expected:**
- Classification emits `domain = DATA`. Status moves to `ROUTED_DATA`.
- `DataSpecialist` returns a `TaskResult` with a SQL query or analysis output in the result body.
- Guardrail allows. Status `COMPLETED`.

## J4 — Ambiguous request: short-circuits to UNROUTABLE

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the ambiguous request to drop.

**Expected:**
- Classification emits `domain = UNKNOWN` with `confidence = low`.
- The workflow terminates immediately with `RequestUnroutable`. Status `UNROUTABLE`.
- No specialist is invoked; the right column shows a muted "Unroutable — no specialist invoked" block.
- `unroutableReason` is populated with the classification reason.
- A `RoutingScored` event still fires; the score chip appears (typically high, since the classifier correctly refused).

## J5 — Guardrail blocks a prompt-injection attempt

**Preconditions:** The seeded JSONL includes one request containing a prompt-injection pattern (e.g. "ignore previous instructions and...").

**Steps:**
1. Wait for the injection-attempt request to drop and be classified.
2. Wait for `RoutingGuardrail` to evaluate it.

**Expected:**
- Status reaches `CLASSIFIED`.
- The guardrail block in the UI shows a red badge with the violation `prompt-injection-detected`.
- Status transitions to `BLOCKED`. No specialist is invoked; `finishedAt` remains null.
- The right column shows the blocked classification and an Unblock button.

## J6 — Routing score appears on every classified request

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new request through the classification step.

**Expected:**
- Within ~10 s of `ClassificationDecided`, the per-request routing score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail.
- The chip never changes the workflow's flow — a low score does not block the result from being published.
