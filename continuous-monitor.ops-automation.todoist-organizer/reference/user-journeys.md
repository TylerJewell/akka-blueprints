# User journeys — todoist-organizer

## J1 — Task arrives and gets classified and updated

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM enabled); `TodoistPoller` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated task.

**Expected:**
- Task appears with status FETCHED, then within 1 s transitions to CLASSIFIED. The classification chip shows the target project name and confidence level.
- Within 1 s the guardrail verdict chip appears showing `allowed: true`.
- Within 2 s the status reaches UPDATED with the update record visible (target project, applied labels, priority level).

## J2 — Low-confidence task is blocked by the guardrail

**Preconditions:** Service running; the simulator includes a canned task with `content: "check that thing"` (no meaningful keywords).

**Steps:**
1. Wait for the low-confidence task to appear in the App UI.
2. Observe the classification chip and guardrail verdict chip.

**Expected:**
- Classification chip shows `confidence: low`.
- Guardrail verdict chip shows `allowed: false` with the reason "Confidence below minimum threshold."
- Status is SKIPPED. The task is not written to the Todoist simulator. The `update` field is absent on the detail panel.

## J3 — Eval score appears on an updated task

**Preconditions:** At least one UPDATED task exists with no `evalScore`. The `EvalRunner` schedule reduced for the test (`EVAL_RUNNER_SECONDS=60`).

**Steps:**
1. Confirm at least one UPDATED task via the task list.
2. Wait 60 s.

**Expected:**
- The task card shows an eval score chip (1–5) and the rationale is visible in the detail panel.
- `GET /api/organizer/{id}` includes `evalScore` populated.

## J4 — Audit log shows guardrail verdict on every task

**Preconditions:** Service running; at least 3 tasks processed (mix of UPDATED and SKIPPED).

**Steps:**
1. Call `GET /api/organizer`.

**Expected:**
- Every completed task (UPDATED, SKIPPED, GUARDRAIL_BLOCKED) has a non-null `guardrailVerdict` object.
- UPDATED tasks have `guardrailVerdict.allowed == true`.
- SKIPPED / GUARDRAIL_BLOCKED tasks have `guardrailVerdict.allowed == false` and a non-empty `reason`.

## J5 — Project allow-list enforces scope

**Preconditions:** Service running; `organizer.project-allowlist` is configured with known project IDs; simulator includes a task the classifier would route to an unlisted project.

**Steps:**
1. Wait for the out-of-scope task to be processed.

**Expected:**
- Even if `confidence` is `"high"`, the guardrail blocks the write because `targetProjectId` is not in the allow-list.
- Status is SKIPPED; `guardrailVerdict.reason` names the allow-list violation.

## J6 — SSE stream delivers real-time updates

**Preconditions:** Service running.

**Steps:**
1. Open `GET /api/organizer/sse` in a browser or with `curl -N`.
2. Wait for a new task to be processed.

**Expected:**
- The stream emits one `organizer-update` event per state transition (FETCHED → CLASSIFIED → UPDATED or SKIPPED).
- Each event's `data` field is a valid JSON `OrganizerTask`.
- No duplicate events for the same `taskId` and same `status`.
