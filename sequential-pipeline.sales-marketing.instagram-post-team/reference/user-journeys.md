# User journeys

Acceptance journeys. Each passes when its expected state changes and UI updates occur.

## J1 — Submit a brief and watch it compose
- **Preconditions:** service running; App UI tab open.
- **Steps:** type a brief, click Submit (`POST /api/briefs`).
- **Expected:** response returns `{ postId }`. Via SSE the post appears in `COMPOSING`, moves to `PROMPTING`, then to `READY` with a non-empty `caption`, non-empty `hashtags`, and non-empty `imagePrompt`, typically within ~30s. The status pill turns green.

## J2 — Blocked content
- **Preconditions:** service running; a brief that drives off-brand or off-policy output.
- **Steps:** submit the brief; let the pipeline run.
- **Expected:** the before-agent-response guardrail fails on the caption or the image prompt; the post transitions to `BLOCKED` with a non-empty `blockReason`; the content is not marked ready; the status pill turns red.

## J3 — Background load from the simulator
- **Preconditions:** service running; no UI interaction.
- **Steps:** wait.
- **Expected:** `BriefSimulator` enqueues a fresh brief every 30s from `sample-events/briefs.jsonl`; `BriefConsumer` starts a workflow per brief; new posts appear in the list on their own and each runs to `READY` or `BLOCKED`.

## J4 — Read a finished post
- **Preconditions:** a post in `READY` (from J1).
- **Steps:** open its card in the App UI.
- **Expected:** the card shows the image prompt, caption, and hashtags together; `safetyCheckPassed` is true; the card carries no block reason.

## J5 — Single-post fetch
- **Preconditions:** a known `postId`.
- **Steps:** `GET /api/posts/{postId}`.
- **Expected:** returns the `PostContent` with its current status and any populated lifecycle fields; `404` for an unknown id.
