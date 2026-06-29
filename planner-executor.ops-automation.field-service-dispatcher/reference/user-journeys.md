# User journeys — field-service-dispatcher

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:<port>/`. App UI tab is visible.
2. In the Work order description field, type "Assign next plumbing work order to the best available technician in Zone 3." Fill in service address "14 Oak St, Zone 3" and required skill "plumbing". Click Submit.
3. A new schedule card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to DISPATCHING via SSE.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - A schedule ledger with a non-empty plan (2–6 steps) and a `currentDispatch` field that is null at completion.
  - A fairness ledger with 2–6 entries. At least one entry has `specialist = ROUTE_OPTIMIZER`, at least one has `specialist = AVAILABILITY`. Every entry has a non-empty `scrubbedResult`.
  - A `DispatchSummary` with a 60–120 word overview and 2–4 assignment bullets.

## J2 — Guardrail blocks an out-of-shift assignment

**Preconditions:** As J1. Fixture technician T-11 has `shiftEnd` set to a past timestamp so the shift is closed.

**Steps:**
1. Submit a work order that the fixture routing logic would initially route to T-11 (based on zone and skill match).

**Expected:**
- The dispatcher's first `DECIDE_ASSIGNMENT` proposes an `AVAILABILITY` check for T-11.
- The guardrail step rejects the decision because T-11's shift window is closed; an `AssignmentBlocked` entry appears on the fairness ledger with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker = "off-shift: technician T-11 shift closed"`.
- The dispatcher replans or proposes a different technician; the schedule either completes via an alternative assignment or fails after the replan budget is exhausted with a clear `failureReason`. Either ending is acceptable; the blocked technician is never dispatched.

## J3 — Operator pause drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any work order that the simulator has not yet seen.
2. While the schedule status is DISPATCHING (within the first ~10 seconds), click **Pause new dispatches** in the operator pane and provide a reason.
3. Observe the in-flight assignment step completes.

**Expected:**
- The in-flight `AssignmentEntry` is recorded normally.
- The next loop iteration reads the pause flag, exits the loop, and emits `SchedulePausedOperator`.
- Schedule status moves to `PAUSED`. `pauseReason` is populated with the operator's reason.
- The operator pane's `PAUSED` pill reflects the state in real time via the `control-update` SSE event.
- Clicking **Resume** clears the pause flag; new work orders submitted after that proceed normally.

## J4 — Credential sanitizer scrubs an exposed key

**Preconditions:** As J1, with the canned fixture file `sample-data/technicians.jsonl` containing one record whose `notes` field holds the literal substring `AKIAIOSFODNN7EXAMPLE`. The `availability.json` mock entry that surfaces this record is exercised on the second or third work order.

**Steps:**
1. Submit "Check availability of all Zone 2 technicians and report their current workloads."

**Expected:**
- The dispatcher dispatches an `AVAILABILITY` subtask.
- `AvailabilityAgent` returns content containing the literal `AKIAIOSFODNN7EXAMPLE`.
- The sanitize step replaces it with `[REDACTED:aws-access-key]` before the entry is recorded.
- `AssignmentEntry.scrubbedResult` does NOT contain `AKIAIOSFODNN7EXAMPLE` anywhere.
- The dispatcher's next prompt sees only the redacted form.
- The final `DispatchSummary.overview` does not contain the literal key.
- The UI's expanded fairness ledger renders the redacted span in italics with a tooltip showing `aws-access-key`.

## J5 — Fairness monitor detects overloaded technician

**Preconditions:** As J1. Submit at least four work orders in the same zone so that fixture routing logic consistently assigns the same technician.

**Steps:**
1. Submit four work orders all requiring `plumbing` in Zone 3 in quick succession.
2. Wait for all four to reach COMPLETED status.
3. Wait up to 5 minutes for the `FairnessMonitor` to tick.

**Expected:**
- `FairnessMonitor` aggregates the fairness ledgers of the four completed schedules.
- The technician assigned to all (or most) of those orders has a share exceeding 1.5× the fleet average.
- A `FairnessAlertRecorded` event fires on one of the schedule entities; `alertType = "overloaded"` and `detail` names the technician id and their assignment share.
- The fairness alerts pane in the App UI tab shows the alert in real time via the `fairness-alert` SSE event.
- The expanded view of the affected schedule shows the alert in the inline fairness alerts section.
