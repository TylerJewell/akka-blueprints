# User journeys — blackboard-swe-coordination

Acceptance criteria. The generated system passes when all six journeys complete as written.

## J1 — Happy path: ticket to MERGED

**Preconditions:** Service running on declared port (9421); a model-provider key set, or the mock LLM selected during scaffolding.

**Steps:**
1. Open `http://localhost:9421/`. The App UI tab is visible.
2. In the form, enter title "Rate-limiter middleware" and a one-line description. Click Submit.
3. A `ticketId` is returned; the board shows the ticket in `INTAKE`.

**Expected:**
- The controller drives the board through each stage: `PLANNED` → `ARCHITECTED` → `CODED` → `REVIEWED` → `VERIFIED` → `INTEGRATION_PLANNED` → `AWAITING_SIGNOFF`. Each transition is visible in the stage pipeline via SSE.
- When the board reaches `AWAITING_SIGNOFF`, an Approve / Reject control appears in the signoff panel.
- Clicking Approve sends `POST /api/signoff/{ticketId}` with `approved: true`.
- The board advances to `MERGED`; the ticket moves to `DONE`.
- The board updates live via SSE throughout, with no page reload.

## J2 — Guardrail rejects a code write with invalid content

**Preconditions:** As J1. Under the mock LLM, `backend-dev.json` includes an entry whose `CodeFile` content contains a `TODO` stub.

**Steps:**
1. Submit a ticket whose mock responses include the stub-containing backend artifact.
2. Watch the board at `ARCHITECTED`.

**Expected:**
- `CodeValidator` rejects the write; `BackendCodeWritten` is never persisted.
- `BlackboardEntity` returns a validation error reply to `ControllerWorkflow`.
- The workflow logs a `guardedReason` and re-invokes `BackendDevAgent` with the validation error text injected into the prompt.
- On the second attempt the mock picks a valid artifact; `BackendCodeWritten` persists and the board advances to `CODED`.
- The validation rejection is visible in the board detail panel.

## J3 — All specialists complete; lead engineer approves

**Preconditions:** As J1, with a ticket that reaches `AWAITING_SIGNOFF`.

**Steps:**
1. Wait for the board to reach `AWAITING_SIGNOFF`.
2. In the signoff panel, enter name "alice" and notes "All checks green." Click Approve.

**Expected:**
- `POST /api/signoff/{ticketId}` with `approved: true, by: "alice"` records the decision on `SignoffEntity`.
- `SignoffWorkflow.finalizeStep` resumes, calls `BlackboardEntity.approve`, and calls `TicketEntity.complete`.
- The board shows `MERGED`; the ticket shows `DONE`.
- `GET /api/signoff/{ticketId}` returns the decision record with `by: "alice"` and `approved: true`.

## J4 — Lead engineer rejects; reviewer re-runs

**Preconditions:** As J3, but the engineer rejects.

**Steps:**
1. Wait for the board to reach `AWAITING_SIGNOFF`.
2. Click Reject with notes "Security finding not addressed."

**Expected:**
- `POST /api/signoff/{ticketId}` with `approved: false` records the rejection.
- The board moves to `IN_REVIEW`.
- `ControllerWorkflow` re-enters from `reviewStep`, invoking `ReviewerAgent` and `SecurityAnalystAgent` again.
- The board advances back toward `AWAITING_SIGNOFF` and the signoff control reappears.

## J5 — Two tickets progress independently

**Preconditions:** As J1.

**Steps:**
1. Submit ticket A with title "CSV export service".
2. Immediately submit ticket B with title "Config hot-reload module".

**Expected:**
- Both tickets appear on the board with separate stage pipelines.
- Each `ControllerWorkflow` instance writes only to its own `BlackboardEntity` (keyed by its `ticketId`).
- No contribution from ticket A appears on ticket B's board, and vice versa.
- Both tickets can reach `AWAITING_SIGNOFF` independently; each requires its own signoff decision.

## J6 — Stuck board alert

**Preconditions:** As J1. The `StuckBoardMonitor` interval is 60 s; the threshold is 5 min.

**Steps:**
1. Submit a ticket and let it advance to any non-terminal stage.
2. Simulate an idle board by waiting 5 minutes (or by reducing the stuck threshold in `application.conf` for testing).

**Expected:**
- `StuckBoardMonitor` detects the idle board and calls `BlackboardEntity.alertStuck`.
- The `BoardStuckAlerted` event persists and the `stuckAlert` field is set on the board row.
- The App UI shows an alert banner on the affected ticket's stage pipeline ("Board idle — possible controller failure").
- The alert does not change the board's stage; an operator must investigate.
