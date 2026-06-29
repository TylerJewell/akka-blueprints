# User journeys — post-incident-reviewer

## J1 — Submit an incident and receive a review ready for sign-off

**Preconditions:** Service running on declared port (`http://localhost:9423/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded incident `INC-2026-0741` has a matching `src/main/resources/sample-data/incidents/INC-2026-0741.json` file with an `IncidentRecord` and ≥ 4 `TimelineEvent` entries.

**Steps:**
1. Open `http://localhost:9423/` → App UI tab.
2. From the **Pick a seeded incident** dropdown, pick `INC-2026-0741`.
3. Click **Start PIR**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `GATHERING` within 1 s more.
- Within ~20 s the card reaches `GATHERED`. The right pane shows the Evidence panel with the incident record fields and a timeline table containing ≥ 4 rows; each row has a non-empty eventId, occurredAt, actor, and description.
- Within ~20 s more the card reaches `ASSESSED`. The Impact Assessment panel shows a severity badge matching the incident's severity, affectedSystems, and a root cause summary with ≥ 1 contributing factor.
- Within ~20 s more the card reaches `DRAFTED`, then `AWAITING_SIGNOFF` within 1 s. The Review panel shows a non-empty executive summary (≥ 50 chars), ≥ 2 action items each with an owner and dueDate, and a timeline whose every eventId is present in the Evidence panel.
- The sign-off panel appears in the right column with an **Approve** and a **Reject** button.
- Total elapsed time to AWAITING_SIGNOFF: ≤ 60 s.

## J2 — Guardrail blocks a draft missing action items

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `draft-review.json` includes one entry with an empty `actionItems` list — the deliberately incomplete entry.

**Steps:**
1. Submit seeded incidents until the fourth submission (the mock selects the incomplete entry on the first iteration of every 4th review by `seedFor(pirId)` modulo).
2. Watch the fourth card's lifecycle in the network panel of the browser dev tools (`/api/pir/sse`).

**Expected:**
- On the fourth review's `draftStep`, the agent's first DRAFT task response has `actionItems: []`. `PIRGuardrail` rejects it; a `GuardrailRejected{check: "action-item-owner", reason: "draft-quality-violation: actionItems list is empty"}` event lands on the entity.
- The incomplete draft NEVER reaches `PIREntity.recordReview` — there is no `ReviewDrafted` event in the log for that iteration.
- The agent's second iteration produces a valid draft with ≥ 1 action item with owner + dueDate. The card eventually reaches `AWAITING_SIGNOFF`.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip shows the one rejected response with its full structured reason.

## J3 — Incident owner approves the review

**Preconditions:** A review is in `AWAITING_SIGNOFF` state (J1 completed).

**Steps:**
1. In the right pane, optionally enter a comment in the comments text area.
2. Click **Approve**.

**Expected:**
- The UI POSTs `{ "approved": true, "comments": "..." }` to `/api/pir/{pirId}/signoff`.
- Within 1 s the card status transitions to `COMPLETE`. The status pill turns green.
- The sign-off decision row appears below the review: decidedBy (from the request), decidedAt, and an approved badge.
- The sign-off panel (approve/reject buttons) disappears.
- A `SignoffRecorded` event and a `ReviewComplete` event are both present in the entity event log.

## J4 — Incident owner rejects the review

**Preconditions:** A review is in `AWAITING_SIGNOFF` state.

**Steps:**
1. In the right pane, enter a rejection comment (e.g., `"Action item owners not from the responsible team — reassign to backend SRE"`).
2. Click **Reject**.

**Expected:**
- The UI POSTs `{ "approved": false, "comments": "Action item owners not from the responsible team — reassign to backend SRE" }` to `/api/pir/{pirId}/signoff`.
- Within 1 s the card status transitions to `REJECTED`. The status pill turns red. The card border highlights red.
- The sign-off decision row shows a rejected badge and the rejection comment.
- A `SignoffRecorded` event and a `ReviewRejected{rejectionComment: "..."}` event are both present in the entity event log.
- The review document itself (the `PostIncidentReview` record) is preserved on the entity — it is not deleted. The UI right pane still shows the review document below the sign-off decision row so the incident owner can reference it when revising.

## J5 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded incident.
2. Wait for `AWAITING_SIGNOFF`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the pirId.

**Expected:**
- The GATHER task's log entries show only `fetchIncidentRecord` and `fetchTimelineEvents` calls.
- The ASSESS task's log entries show only `classifyImpact` and `identifyRootCause` calls.
- The DRAFT task's log entries show only `composeExecutiveSummary` and `buildActionItems` calls.
- No cross-phase calls appear. The order of tasks in the log is GATHER → ASSESS → DRAFT. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J6 — Unknown incident ID (no sample data)

**Preconditions:** Service running. No `src/main/resources/sample-data/incidents/<custom-id>.json` exists for the entered ID.

**Steps:**
1. In the App UI, type a custom incident ID (e.g., `INC-9999-9999`) into the incident ID field.
2. Click **Start PIR**.

**Expected:**
- `GatherTools.fetchIncidentRecord` returns null / throws a not-found response.
- The agent's GATHER task returns an `EvidenceLog` with an empty timeline (or the minimal incident stub the tool returns for unknown IDs).
- The workflow advances to `assessStep`. The ASSESS task produces an `ImpactAssessment` noting "insufficient evidence."
- The workflow advances to `draftStep`. The DRAFT task produces a `PostIncidentReview` with `executiveSummary = "(no evidence gathered — review cannot be completed)"` and `actionItems = []`.
- `PIRGuardrail` accepts the response (the refusal path is exempt from the 50-char minimum and action-item requirement when the summary matches the exact refusal text).
- The card reaches `AWAITING_SIGNOFF` with an honest empty review; nothing crashes; the incident owner can reject it with a comment asking for the correct incident ID.
