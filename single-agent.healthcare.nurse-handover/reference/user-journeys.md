# User journeys — nurse-handover

## J1 — Submit a General Medicine handover and get a summary

**Preconditions:** Service running on declared port (`http://localhost:9170/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9170/` → App UI tab.
2. From the **Ward** dropdown, pick `General Medicine`.
3. Click **Load seeded example** to fill the shift report textarea.
4. Fill in `Submitted by` with any name and click **Submit handover**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted report; PHI category chips show `mrn` and `dob` (at minimum).
- Within 30 s the card reaches `SUMMARY_READY` then immediately `AWAITING_SIGNOFF`. The right pane shows: a per-patient status table with one row per patient in the seed report, an outstanding-tasks list ordered by urgency, any risk flags, and the narrative summary paragraph.
- The **Sign off** button is visible on the card in the live list.

## J2 — Incoming clinician signs off and the handover is sealed

**Preconditions:** J1 has completed; a handover is in `AWAITING_SIGNOFF` state.

**Steps:**
1. In the right-pane detail, fill in the `Signed off by` field with a clinician name (different from the submitter).
2. Click **Sign off**.
3. Observe the card in the live list.

**Expected:**
- The card transitions from `AWAITING_SIGNOFF` to `SIGNED_OFF` within 2 s (one poll cycle of `awaitSignoffStep`).
- The right-pane detail shows `Signed off by: <name> at <time>` in the sign-off section; the **Sign off** button disappears.
- A second click on a now-absent button (or a direct `PATCH /api/handovers/{id}/signoff` API call) returns `409 Conflict`. No second `HandoverSignedOff` event is emitted.

## J3 — PHI never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a report containing the literal strings `MRN 789-45-6123`, `DOB 1962-03-14`, and `Tel 555-0199`.
2. Wait for `SUMMARY_READY`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/handovers/{id}` and read `request.rawReport`.

**Expected:**
- The logged LLM call body contains the redacted forms only: `[REDACTED-MRN]`, `[REDACTED-DOB]`, `[REDACTED-PHONE]`. The raw strings do not appear in the agent's attachment.
- `request.rawReport` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.phiCategoriesFound` lists `mrn`, `dob`, `phone` (at minimum).

## J4 — ICU report surfaces CRITICAL risk flags and urgency ordering

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the Ward dropdown, pick `ICU`.
2. Load the seeded ICU example and submit.
3. Wait for `AWAITING_SIGNOFF`.

**Expected:**
- The summary's `riskFlags` list is non-empty and contains at least one flag mentioning the ICU-specific risk condition in the seed (e.g., vasopressor titration or ventilator weaning).
- The `outstandingTasks` list has all CRITICAL items before any URGENT or ROUTINE items.
- At least one `PatientStatus` row shows `riskLevel: CRITICAL` or `riskLevel: HIGH`.

## J5 — Checklist item with no corresponding report note is surfaced

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit any seed report using the General Medicine ward checklist.
2. Manually edit the submitted checklist to include a checklist item whose `itemId` is not addressed anywhere in the seed report (e.g., `gm-family-notification`).
3. Submit and wait for `AWAITING_SIGNOFF`.

**Expected:**
- The relevant patient's `outstandingTasks` string includes the undocumented checklist item marked as "not documented".
- The ward-level `outstandingTasks` list includes an entry for the undocumented item.
- The summary is still well-formed (the agent does not refuse or omit the item).

## J6 — Handover in AWAITING_SIGNOFF remains accessible after service restart

**Preconditions:** Service running. One handover is in `AWAITING_SIGNOFF` state (J1 completed, J2 not yet done).

**Steps:**
1. Stop the service (Ctrl-C or `/akka:build` stop).
2. Restart the service.
3. Open the UI and navigate to the App UI tab.

**Expected:**
- The handover in `AWAITING_SIGNOFF` is visible in the live list with its status, ward, and summary intact.
- The **Sign off** button is still present and functional — completing J2 after restart produces the same `SIGNED_OFF` result.
- The `HandoverWorkflow`'s `awaitSignoffStep` resumes its polling without manual intervention, within one poll cycle (5 s) of restart.
