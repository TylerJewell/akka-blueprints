# User journeys — lambda-error-diagnoser

## J1 — Error log arrives and gets diagnosed

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM configured); `LogPoller` enabled.

**Steps:**
1. Open `http://localhost:9283/` → App UI tab.
2. Wait up to 20 s for the first simulated error event.

**Expected:**
- Incident appears with status DETECTED, transitions to NORMALISED within 1 s. The normalised error category and severity badge are visible.
- Within 30 s the diagnosis arrives: root cause, fix suggestion, and confidence chip are shown.
- Immediately after, the eval score chip (1–5) appears on the same incident card.
- Status reaches RESOLVED. The full sequence DETECTED → NORMALISED → DIAGNOSED → EVALUATED → RESOLVED completes within 60 s total.

## J2 — Operator dismisses a false-positive incident

**Preconditions:** At least one RESOLVED incident is visible on the board.

**Steps:**
1. Select the incident in the list.
2. Click Dismiss. A reason textarea opens.
3. Type a reason (e.g., "Flapping test Lambda — not production") and click Confirm.

**Expected:**
- Status transitions to DISMISSED immediately. The reason is visible in the detail pane.
- The incident remains on the board (not deleted) with a DISMISSED badge.
- `GET /api/incidents/{id}` returns the incident with `dismissReason` populated.

## J3 — Operator re-opens a dismissed incident

**Preconditions:** A DISMISSED incident is visible on the board.

**Steps:**
1. Select the DISMISSED incident.
2. Click Re-open.

**Expected:**
- Status transitions to REOPENED. The incident card updates to show the REOPENED badge.
- The original diagnosis and eval score remain visible — they are not cleared on reopen.
- `GET /api/incidents/{id}` returns `status: "REOPENED"`.

## J4 — Every RESOLVED incident carries an eval score

**Preconditions:** At least three RESOLVED incidents exist (wait 60 s after start).

**Steps:**
1. Call `GET /api/incidents?status=RESOLVED`.

**Expected:**
- Every item in the response array has a non-null `eval` field with `score` between 1 and 5.
- No item has `eval: null` or `eval.score: 0`.

## J5 — High-severity incident surfaced at the top of the board

**Preconditions:** Service running for at least 40 s so that multiple incidents of different severities have been processed.

**Steps:**
1. Open the App UI tab.
2. Inspect the incident list without scrolling.

**Expected:**
- CRITICAL and HIGH severity incidents appear above MEDIUM and LOW incidents regardless of arrival order.
- Severity badges (CRITICAL=red, HIGH=orange, MEDIUM=yellow, LOW=muted) are visible on each card.
