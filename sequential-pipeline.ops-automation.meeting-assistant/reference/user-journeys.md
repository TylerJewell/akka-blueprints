# User journeys

Each journey lists preconditions, steps, expected state changes, expected UI updates, and the pass condition.

## J1 — Submit notes and watch action items extract

- **Preconditions:** service running; a model key resolved (or mock mode).
- **Steps:** in the App UI tab, paste meeting notes, click Submit.
- **Expected state:** `RECEIVED → SANITIZED → EXTRACTED`. `summary` non-empty; `actions` has at least one item.
- **Expected UI:** the meeting card appears, advances through status chips, and lists each action item with its assignee hint.
- **Pass:** within ~30s the meeting shows `EXTRACTED` with a summary and one or more action items.

## J2 — Personal data is redacted before extraction

- **Preconditions:** as J1; submit notes containing an email and a phone number.
- **Steps:** submit; open the meeting card.
- **Expected state:** `redactionCount > 0`; `sanitizedNotes` contains no raw email or phone string.
- **Expected UI:** the card shows the redaction count once `SANITIZED`.
- **Pass:** the agent's output references redaction tokens, not the original personal data, and `redactionCount` matches the number of redactions.

## J3 — Action items fan out to Trello and Slack

- **Preconditions:** J1 reached `EXTRACTED`; target board and channel are allowlisted (default config); no real credentials needed.
- **Steps:** none — `dispatchStep` runs automatically.
- **Expected state:** `DISPATCHED`; each `ActionItem` has a `trelloCardUrl`; `slackMessageTs` non-empty.
- **Expected UI:** each action item shows a Trello card link; the card shows the Slack timestamp.
- **Pass:** one Trello card per action item and exactly one Slack message; re-running dispatch creates no duplicates (idempotency key).

## J4 — Guardrail refuses a non-allowlisted target

- **Preconditions:** configure a meeting whose target board or channel is not on the allowlist.
- **Steps:** submit; let it reach `EXTRACTED`.
- **Expected state:** `FAILED` with a `failureReason` naming the refused target; no POST reaches `SimEndpoint`.
- **Expected UI:** the card shows `FAILED` and the refusal reason.
- **Pass:** the before-tool-call guardrail blocks the write; the meeting is `FAILED`; `SimEndpoint` received nothing for that meeting.

## J5 — Background load from the simulator

- **Preconditions:** service running, no UI interaction.
- **Steps:** wait.
- **Expected state:** `NotesSimulator` enqueues a canned note every 30s; each runs the full pipeline.
- **Expected UI:** new meetings appear on their own.
- **Pass:** meetings accumulate without any manual submission.
