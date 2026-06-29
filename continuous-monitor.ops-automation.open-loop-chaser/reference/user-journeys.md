# User journeys — open-loop-chaser

## J1 — Source event is ingested and action items appear

**Preconditions:** Service running on declared port; valid model-provider API key set; `ActionItemPoller` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 20 s for the first simulated source event.

**Expected:**
- Within 1 s of the source event being ingested, the item list shows one or more entries with status DETECTED, then SANITIZED.
- Within 30 s the status transitions to PENDING. The sanitized description is visible; no raw names or email addresses appear in the UI.
- The source type chip (MEETING_NOTES, MESSAGE, or DOCUMENT) is shown on the card.

## J2 — Owner confirmed; stale item triggers a nudge

**Preconditions:** At least one item in PENDING state.

**Steps:**
1. Select the item in the list.
2. Enter a confirmed owner name in the Confirm Owner field; click Confirm.
3. Wait for `StaleItemScanner` to tick (default 10 minutes; reduce `SCANNER_INTERVAL_SECONDS` for the test).

**Expected:**
- The item transitions to STALE after the scanner tick.
- `NudgeWorkflow` starts; `NudgeComposerAgent` drafts a nudge. The guardrail passes because `ownerConfirmation` is present.
- The item transitions to NUDGED. The nudge count increments to 1. The composed nudge appears in the nudge history section of the detail panel.
- No real network call leaves the process (the `sendNudge` tool is a no-op stub).

## J3 — Snoozing an item suppresses nudges

**Preconditions:** A PENDING item in the list.

**Steps:**
1. Select the item.
2. Click Snooze; set duration to 60 minutes; confirm.

**Expected:**
- Status transitions to SNOOZED immediately.
- During the snooze window, `StaleItemScanner` ticks skip this item — no STALE transition, no NudgeWorkflow started.
- After the snooze duration elapses (or after `SnoozeLapsed` is emitted manually via test), the item returns to PENDING and re-enters the normal staleness check cycle.

## J4 — Closing an item stops all future nudges

**Preconditions:** A NUDGED item in the list.

**Steps:**
1. Select the item.
2. Click Close; enter a closure note; confirm.

**Expected:**
- Status transitions to CLOSED immediately. `finishedAt` is populated.
- Subsequent `StaleItemScanner` ticks skip this item entirely.
- The item card transitions to terminal styling (muted status pill). No further nudge count increments.

## J5 — Guardrail blocks nudge for unconfirmed item

**Preconditions:** A STALE item whose `ownerConfirmation` is null (owner was never confirmed).

**Steps:**
1. Allow `StaleItemScanner` to mark the item STALE without confirming the owner.
2. Observe the `NudgeWorkflow` execution in the service logs.

**Expected:**
- `NudgeComposerAgent` composes a nudge (compose step succeeds).
- The guardStep reads the entity and finds `ownerConfirmation.isEmpty()`.
- The guardrail rejects the `sendNudge` call. An `ExtractionFailed` event is emitted with `reason="guardrail:no-confirmed-owner"`.
- The item transitions to FAILED. The service log contains a line recording the block with the item id.
- No nudge appears in the item's nudge history; `nudgeCount` remains 0.

## J6 — PII does not reach the LLM (audit check)

**Preconditions:** Service running with debug logging enabled (`LOG_LEVEL=DEBUG`).

**Steps:**
1. Submit a custom source event via `POST /api/items` source simulation with text `"alex.smith@example.com will complete the review by Friday"`.
2. Inspect the service log for the LLM call payload.

**Expected:**
- The log shows the LLM call body contains `[REDACTED-EMAIL]` — never `alex.smith@example.com`.
- The `SourceEventQueue`'s `SourceEventReceived` event (the audit log) still has the raw text; the `ActionItemEntity`'s `sanitized.redactedText` has the redacted form.
