# User journeys — scrum-master-bot

## J1 — Run a standup for the feature team fixture

**Preconditions:** Service running on declared port (`http://localhost:9181/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9181/` → App UI tab.
2. From the **Sprint fixture** dropdown, pick `Feature Team (3 members, sprint 12)`.
3. The session panel pre-fills with the seeded roster (Alice, Bob, Carol) and 6 authorized ticket ids (FEAT-10 through FEAT-14, BUG-42).
4. Click **Run standup**.

**Expected:**
- The new session card appears in the live list with status `COLLECTING` within 1 s.
- The card transitions to `RUNNING` within 1 s. The right-pane detail shows the sprint context (sprint name, dates, member count).
- Within 60 s the card reaches `SUMMARY_READY`. The right pane shows: an outcome badge (ON_TRACK / AT_RISK / BLOCKED), a summary paragraph, and a per-member row for each of the 3 team members. Every member row has non-empty `yesterday` and `today` text.
- Within 1 s of `SUMMARY_READY`, the card reaches `POSTED`. The ticket updates section shows a green posted badge for each ticket in the `ticketUpdates` list.

## J2 — Guardrail blocks an out-of-scope ticket write

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `conduct-standup.json` includes entries whose `ticketUpdates` reference ticket ids not in the sprint's `authorizedTicketIds`.

**Steps:**
1. Start a standup with the Platform Team fixture (4 members, sprint 5, 8 authorized tickets).
2. Watch the session lifecycle in the network panel of the browser dev tools (`/api/standups/sse`).

**Expected:**
- The agent's task includes a `postTicketUpdate` call for a ticket id not in the authorized set (e.g., `FEAT-99`).
- The `before-tool-call` guardrail rejects the call. The ticket is never posted — there is no successful post log entry for `FEAT-99`.
- The agent continues within its iteration budget. The final `StandupSummary.ticketUpdates` list either omits the rejected ticket or includes it with a `newStatus = null` and the rejection noted in `nextActions`.
- The service log shows one `guardrail.reject` line with the error code `out-of-scope-ticket: FEAT-99`.
- The `postResult.skippedTicketIds` list on the session includes `FEAT-99`.

## J3 — All members report a blocker → BLOCKED outcome

**Preconditions:** Mock LLM mode. The mock response selected for this session returns `MemberUpdate` entries where all members have a non-null `blocker` field.

**Steps:**
1. Start a standup with the Release Management fixture (5 members, sprint 3, 10 authorized tickets).
2. Wait for `SUMMARY_READY`.

**Expected:**
- All 5 `memberUpdates` entries have a non-null `blocker` string.
- `sessionOutcome` is `BLOCKED`.
- The session card in the live list shows a red outcome badge.
- The right-pane `nextActions` list contains at least one item beginning with "Escalate" or "Unblock".

## J4 — Sprint fixture update changes the authorized scope

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Run a standup with the Feature Team fixture (authorized tickets: FEAT-10 through FEAT-14, BUG-42). Note the 6-ticket scope.
2. Before running the next standup, edit the Sprint fixture panel: remove `BUG-42` from the authorized ticket ids and add `FEAT-15`.
3. Click **Run standup** again.

**Expected:**
- The new session's `sprintContext.authorizedTicketIds` shows the updated 6-ticket set (FEAT-10 through FEAT-15, no BUG-42).
- If the agent attempts to post to `BUG-42`, the guardrail rejects it. Any post to `FEAT-15` succeeds.
- The two sessions in the live list show distinct sprint contexts — the old session retains the original scope, the new session reflects the updated one.

## J5 — Session fails gracefully and preserves partial data

**Preconditions:** Mock LLM mode configured to return a timeout or malformed response on the third standup attempt.

**Steps:**
1. Run two successful standups (J1 steps twice).
2. Run a third standup. The mock provider is configured to exhaust the agent's iteration budget on the third attempt.

**Expected:**
- The session card transitions to `FAILED` status (red status pill).
- The right pane shows the partial data collected before failure: sprint context is populated (it was attached before the agent step), but `summary` is `null` and `postResult` is `null`.
- The service log shows the workflow's `error` step firing and `SessionFailed` emitted to `StandupEntity`.
- The prior two sessions remain in `POSTED` status; they are unaffected.
