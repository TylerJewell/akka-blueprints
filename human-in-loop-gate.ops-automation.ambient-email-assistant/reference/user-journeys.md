# User journeys

Acceptance journeys the generated system must pass.

## J1 — Triage an incoming email

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/email-threads` with `{ "sender": "...", "subject": "Question about Q3 deliverables", "body": "Hi, could you share the status of the Q3 report?" }`.
- **Expected:** response returns `{ threadId }`. Within ~30 s the thread appears via SSE in `TRIAGED` with `category=inquiry`, a non-empty `urgency`, and `suggestedAction=reply`. A draft action then appears and the thread transitions to `ACTION_DRAFTED` with non-empty `draftSubject` and `draftBody`. Approve/Dismiss buttons appear on the UI card.

## J2 — Approve and send a reply

- **Preconditions:** a thread in `ACTION_DRAFTED` with `suggestedAction=reply` (J1).
- **Steps:** `POST /api/threads/{threadId}/approve` with `{ "approvedBy": "ops-team", "note": "LGTM" }`.
- **Expected:** `EmailThreadEntity` transitions to `APPROVED`; within ~30 s status becomes `REPLY_SENT` with a non-null `actionReference` (e.g., `MSG-<uuid>`). The UI card reflects the terminal state and hides the Approve/Dismiss buttons.

## J3 — Approve and schedule a meeting

- **Preconditions:** a thread in `ACTION_DRAFTED` with `suggestedAction=meeting`. Submit an email like `{ "subject": "Let's sync on the roadmap", "body": "Can we meet Thursday at 2 PM UTC?" }`.
- **Steps:** `POST /api/threads/{threadId}/approve` with an approver name.
- **Expected:** `EmailThreadEntity` transitions to `APPROVED`; within ~30 s status becomes `MEETING_SCHEDULED` with a non-null `actionReference` (e.g., `EVT-<uuid>`). The UI card shows the proposed meeting time and attendees.

## J4 — Dismiss a thread

- **Preconditions:** a thread in `ACTION_DRAFTED` (J1 or J3 precondition met).
- **Steps:** `POST /api/threads/{threadId}/dismiss` with `{ "dismissedBy": "ops-team", "note": "Not relevant" }`.
- **Expected:** status becomes terminal `DISMISSED`; the dismiss note is shown on the UI card; no Gmail or Calendar tool call fires. The workflow ends.

## J5 — Action guard blocks unapproved tool call

- **Preconditions:** a thread in `ACTION_DRAFTED` (not yet approved).
- **Steps:** observe the workflow poll cycle while status is still `ACTION_DRAFTED`.
- **Expected:** the before-tool-call guardrail re-reads `EmailThreadEntity.status`; because it is not `APPROVED`, the simulated Gmail send and Calendar create tools do not run. The thread stays in `ACTION_DRAFTED` until the operator acts.

## J6 — Metadata tabs render correctly

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 4 controls (H1, G1, S1, E1) in `matrix-row` style with mechanism pills; `hitl` pill is yellow, `guardrail` is red, `sanitizer` is blue, `eval-event` is purple. Risk Survey renders pre-filled answers with `TO_BE_COMPLETED_BY_DEPLOYER` placeholders muted italic.
