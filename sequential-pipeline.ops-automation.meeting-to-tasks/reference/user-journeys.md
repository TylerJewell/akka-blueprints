# User journeys — meeting-to-tasks

Acceptance journeys. Passing all of these means the blueprint generated correctly.

## J1 — Submit notes and watch tasks extract

- **Preconditions**: service running on `http://localhost:9450/`; a model key set or the mock provider selected.
- **Steps**: POST `/api/meetings` with `{ "title": "Weekly sync", "rawNotes": "..." }`. Subscribe to `/api/meetings/sse`.
- **Expected state**: the meeting moves `RECEIVED → SANITIZED → EXTRACTED → AWAITING_APPROVAL` within ~30s, ending with a non-empty `extractedTasks` list and `redactionCount` present.
- **Expected UI**: the meeting appears in the live list; the task list renders with Approve/Reject buttons.
- **Done when**: status is `AWAITING_APPROVAL` and `extractedTasks` is non-empty.

## J2 — Approve and watch the outputs complete

- **Preconditions**: a meeting in `AWAITING_APPROVAL` (from J1).
- **Steps**: POST `/api/meetings/{id}/approve` with `{ "approvedBy": "reviewer" }`.
- **Expected state**: the meeting moves `APPROVED → COMPLETED` within ~30s with non-null `trelloBoardUrl`, `trelloCardCount`, `csvPath`, and `slackMessageTs`.
- **Expected UI**: the Approve/Reject buttons disappear; the row shows the board URL, CSV path, and Slack reference.
- **Done when**: status is `COMPLETED` with all four output fields populated.

## J3 — Reject and confirm no external write

- **Preconditions**: a meeting in `AWAITING_APPROVAL`.
- **Steps**: POST `/api/meetings/{id}/reject` with `{ "reason": "out of scope" }`.
- **Expected state**: the meeting moves to `REJECTED`; `trelloBoardUrl`, `csvPath`, and `slackMessageTs` stay null.
- **Expected UI**: the row shows the reject reason and no output references.
- **Done when**: status is `REJECTED` and no output field is set.

## J4 — PII is redacted before any external write

- **Preconditions**: service running.
- **Steps**: POST `/api/meetings` with `rawNotes` containing an email (`alex@example.com`) and an attendee name. Approve the resulting meeting.
- **Expected state**: `sanitizedNotes` contains neither the email nor the name; `redactionCount > 0`; the payloads sent to the board and chat (visible in the in-process facade log or the real services) carry no un-redacted email or name.
- **Expected UI**: the redaction count shows on the row.
- **Done when**: the sanitized notes and the outbound payloads are clean and `redactionCount > 0`.

## J5 — Simulator seeds a meeting with no UI interaction

- **Preconditions**: service running; no manual submit.
- **Steps**: wait up to 45s.
- **Expected state**: `NotesSimulator` drips a canned meeting from `meeting-notes.jsonl`; a fresh meeting runs the pipeline to `AWAITING_APPROVAL` on its own.
- **Expected UI**: a meeting the user did not submit appears in the live list.
- **Done when**: an un-submitted meeting reaches `AWAITING_APPROVAL`.
