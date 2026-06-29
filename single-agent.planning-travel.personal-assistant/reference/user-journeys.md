# User journeys — personal-assistant

## J1 — Submit a travel request and get an action

**Preconditions:** Service running on declared port (`http://localhost:9559/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9559/` → App UI tab.
2. From the **Context profile** dropdown, pick `Travel week`.
3. Click **Load example** to fill the context snapshot.
4. In the **Request** textarea, type: `Schedule a hotel check-in reminder for Monday at 9 AM`.
5. Click **Send request**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted context JSON; the PII category chips show at minimum `address` (from the hotel location field in the Travel week profile).
- Within 30 s the card reaches `ACTION_RECORDED`. The right pane shows: an action type badge (`SET_REMINDER` or `CREATE_TASK`), the explanation paragraph, and a change-set table with at least one non-empty row. Every `newValue` in the change set is non-empty.
- Within 1 s of `ACTION_RECORDED`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a past-date write

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `handle-request.json` includes a deliberately malformed entry with a `date` value in the past.

**Steps:**
1. Submit any seeded request three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/requests/sse`).

**Expected:**
- The third submission's first agent iteration proposes a `CREATE_EVENT` with a date in the past.
- The `before-tool-call` guardrail rejects it. The malformed action NEVER lands in `AssistantEntity` — there is no `ActionRecorded` event with the past-date payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed action with a future date. The card transitions to `ACTION_RECORDED` with a valid change set.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed (`date-in-past`).

## J3 — Empty change set flags eval score 1

**Preconditions:** Mock LLM mode. The mock returns a `NO_ACTION` response for a request that clearly demands an action.

**Steps:**
1. In the App UI, type a request that maps to the mock's deliberate `NO_ACTION` entry (e.g. "Add a reminder to check the flight status tomorrow morning").
2. Submit with the Minimal context profile.

**Expected:**
- The action lands well-formed (the guardrail only checks write validity, not action relevance).
- The eval score chip shows **1** and the rationale reads something like "Action type is NO_ACTION but the request demanded a create or set-reminder; change set is empty."
- The card's border highlights red. The user knows to resubmit or adjust the request.

## J4 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a request with a context snapshot containing `alice@example.com`, `+1-555-0100`, and `12 Elm Street, Springfield` in a task description.
2. Wait for `ACTION_RECORDED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/requests/{id}` and read `request.rawContext`.

**Expected:**
- The logged LLM call body contains the redacted forms only: `[REDACTED-EMAIL]`, `[REDACTED-PHONE]`, `[REDACTED-ADDRESS]`. The raw strings do not appear.
- `request.rawContext` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.piiCategoriesFound` lists at minimum `email`, `phone`, `address`.

## J5 — COMPLETE_TASK action targets the correct task

**Preconditions:** Service running. Any model provider. Travel week context loaded (contains task `task-001` "Book airport transfer" with `completed: false`).

**Steps:**
1. Submit the request: `Mark the airport transfer task as done`.
2. Wait for `EVALUATED`.

**Expected:**
- The action's `actionType` is `COMPLETE_TASK`.
- The change set contains a `ChangeField` with `field = "targetId"` and `newValue = "task-001"`, and a `ChangeField` with `field = "completed"`, `oldValue = "false"`, `newValue = "true"`.
- The eval score is 4 or 5 (specific target, non-empty change set, action type matches intent).

## J6 — Duplicate prevention via UPDATE over CREATE

**Preconditions:** Service running. Travel week context already contains an event `evt-002` "Hotel check-in" on Monday.

**Steps:**
1. Submit: `Add a hotel check-in event for Monday`.
2. Wait for `EVALUATED`.

**Expected:**
- The agent detects the existing `evt-002` event in the context attachment.
- The action's `actionType` is `UPDATE_TASK` or `SET_REMINDER` targeting `evt-002`, not `CREATE_EVENT` for a duplicate entry.
- The change set references `targetId = "evt-002"`.
- The eval score is 4 or 5; a score of 3 or below with a rationale mentioning "duplicate" is also acceptable if the mock returns a `CREATE_EVENT` (the guardrail does not block duplicates — that is an eval signal, not a hard block).
