# User journeys — nexshift

## J1 — Submit a 5-shift nursing week and confirm

**Preconditions:** Service running on declared port (`http://localhost:9451/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9451/` → App UI tab.
2. From the **Department** dropdown, pick `Nursing`.
3. Click **Load seeded example** to fill the shift list and employee roster.
4. Click **Generate schedule**.

**Expected:**
- The new run card appears in the live list with status `PENDING` within 1 s.
- The card transitions to `BUILDING` within 1 s.
- Within 60 s the card reaches `DRAFT`. The right-pane assignment table shows all 5 shifts with status `ASSIGNED`. The filled chip reads `5 / 5`. No `GUARDRAIL_BLOCKED` rows.
- The manager clicks **Confirm schedule**. Within 1 s the card transitions to `CONFIRMED` and the confirm button disappears.

## J2 — Guardrail blocks a double-booking attempt

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `assign-shifts.json` includes an entry where the agent first calls `AssignShift(shift-002, emp-101)` while `emp-101` is already assigned to `shift-001` (overlapping window).

**Steps:**
1. Submit the seeded nursing week (J1 steps 2–4).
2. Open the browser network panel and watch `/api/schedules/sse`.

**Expected:**
- The SSE stream shows `SCHEDULING` state.
- The service log shows one `guardrail.reject` line for `AssignShift(shift-002, emp-101)` with reason `overlap-detected`.
- The agent retries with `emp-102` and the call is accepted.
- The draft's assignment table shows `shift-002` assigned to `emp-102` with status `ASSIGNED`, not `emp-101`.
- There is no `VerdictRecorded` equivalent with the invalid assignment in the entity log.

## J3 — Hour-cap enforcement blocks an over-scheduled employee

**Preconditions:** Service running (real or mock LLM). The seeded employee roster includes `emp-103` with `weeklyHoursCap = 40` and `hoursScheduledThisWeek = 40`.

**Steps:**
1. Submit a scheduling run that includes `emp-103` in the roster.
2. Wait for `DRAFT`.

**Expected:**
- Every `AssignShift` call targeting `emp-103` is rejected with reason `hour-cap-exceeded`.
- `emp-103` appears in the assignment table zero times with status `ASSIGNED`.
- The guardrail audit log (service log `guardrail.reject`) shows one entry per rejected call naming `employeeId = emp-103` and `check = hour-cap-exceeded`.
- All shifts that had an alternative eligible employee are filled; shifts with no alternative appear as `UNASSIGNED`.

## J4 — Qualification check prevents unqualified placement

**Preconditions:** Service running (real or mock LLM). Shift `shift-003` requires qualification `cert-ACLS`. Employee `emp-104` holds only `cert-BLS`.

**Steps:**
1. Submit a scheduling run containing `shift-003` and a roster where `emp-104` is the only initially attempted candidate.
2. Wait for `DRAFT`.

**Expected:**
- `AssignShift(shift-003, emp-104)` is rejected with reason `qualification-mismatch` and detail `missing: cert-ACLS`.
- The agent tries the next employee in rank order.
- If an employee with `cert-ACLS` is available, `shift-003` is assigned to them.
- The service log confirms the guardrail fired for the unqualified attempt.

## J5 — Live SSE view tracks transitions without a page refresh

**Preconditions:** Service running. Two browser tabs open to the App UI.

**Steps:**
1. In Tab A, submit a scheduling run.
2. Do not interact with Tab B.

**Expected:**
- Tab B's live list updates automatically as the run progresses through `BUILDING → SCHEDULING → DRAFT` without a page reload.
- The filled chip in Tab B updates from `0 / 5` to `5 / 5` as assignments land.
- When the manager confirms in Tab A, Tab B shows `CONFIRMED` within 1 s.
