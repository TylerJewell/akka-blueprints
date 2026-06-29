# User journeys — meeting-facilitator

## J1 — Session arrives and gets published

**Preconditions:** Service running on declared port; valid model-provider API key set; `MeetingSessionPoller` enabled.

**Steps:**
1. Open `http://localhost:9316/` → App UI tab.
2. Wait up to 20 s for the first simulated segment.

**Expected:**
- Session appears with status ACTIVE, then within 1 s transitions to SANITIZED. The sanitized transcript snippet is visible in the card; the raw speaker name is not.
- PII category chips appear (e.g., "name") on the session card.
- Within 60 s the status reaches PUBLISHED. The meeting notes and chat recap are visible in the detail pane. The guardrail verdict badge shows PASS.

## J2 — PII is redacted before the LLM sees the transcript

**Preconditions:** Service running with debug logging enabled (`LOG_LEVEL=DEBUG`).

**Steps:**
1. Submit a custom segment via the poller's JSONL feed with `speakerRaw` set to "Jane Smith" and `textRaw` containing "Jane Smith's email is jsmith@example.com".
2. Inspect the service log for the LLM call payload.

**Expected:**
- The log shows the `NotesSummaryAgent` call body contains `[REDACTED-NAME]` and `[REDACTED-EMAIL]` — never "Jane Smith" or "jsmith@example.com".
- The entity's `rawSegment.textRaw` (the audit log) still holds the original values.
- The entity's `sanitized.piiCategoriesFound` includes `"name"` and `"email"`.

## J3 — Guardrail suppresses a policy-violating summary

**Preconditions:** The sample JSONL file includes a segment labelled `guardrail-trip` (pre-seeded with content the guardrail rules flag).

**Steps:**
1. Wait for the `guardrail-trip` segment to be processed (within 60 s of the poller reaching that line).

**Expected:**
- Status transitions to SUPPRESSED rather than PUBLISHED.
- The session card shows the red FAIL badge and the guardrail `reason` string.
- No `SessionPublished` event is emitted; `SessionSuppressed` is present in the entity event log.

## J4 — Eval score appears on a published session

**Preconditions:** At least one PUBLISHED session exists with no `evalScore`. The `EvalRunner` schedule reduced for the test (`EVAL_RUNNER_SECONDS=60`).

**Steps:**
1. Confirm a session reaches PUBLISHED.
2. Wait 60 s.

**Expected:**
- The session card shows an eval score chip (1–5) and the eval rationale is visible in the detail pane.
- The response from `GET /api/sessions/{id}` includes `evalScore` populated.

## J5 — SSE pushes real-time updates to the UI

**Preconditions:** Service running; browser open on App UI tab.

**Steps:**
1. Open the browser's developer tools → Network → filter by `text/event-stream`.
2. Watch for the next `session-update` event.

**Expected:**
- Each state transition (ACTIVE → SANITIZED → SUMMARIZED → PUBLISHED) emits a distinct SSE event with the updated `MeetingSession` JSON.
- The UI card updates without a page reload, driven by the EventSource client.
