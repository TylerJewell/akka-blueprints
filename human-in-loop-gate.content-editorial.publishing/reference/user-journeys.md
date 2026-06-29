# User journeys

Acceptance journeys the generated system must pass.

## J1 — Draft a post

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/publish-request` with `{ "topic": "..." }`.
- **Expected:** response returns `{ postId }`. Within ~30 s the post appears via SSE in `DRAFTED` with non-empty `draftTitle` and `draftContent`. Approve/Reject buttons show on the App UI card.

## J2 — Approve and publish

- **Preconditions:** a post in `DRAFTED` with content (J1).
- **Steps:** `POST /api/posts/{postId}/approve` with an approver name.
- **Expected:** `PostEntity` transitions to `APPROVED`; the workflow's next poll moves to `publishStep`; `PublishingAgent` runs; within ~30 s status becomes `PUBLISHED` with a non-null `publishedUrl` shown in the UI.

## J3 — Reject a draft

- **Preconditions:** a post in `DRAFTED` (J1).
- **Steps:** `POST /api/posts/{postId}/reject` with a reason.
- **Expected:** status becomes terminal `REJECTED`; the reason is shown; the workflow ends; the publish step never runs.

## J4 — Publish guard blocks premature publish

- **Preconditions:** a post in `DRAFTED` (not yet approved).
- **Steps:** drive the workflow toward `publishStep` without an approval (e.g., the await-approval poll runs while status is still `DRAFTED`).
- **Expected:** the before-tool-call guardrail re-reads `PostEntity.status`; because it is not `APPROVED`, the simulated publish tool does not run and no `publishedUrl` is set. The post stays in `DRAFTED` until a human acts.

## J5 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, G1, G2) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted; the README/Overview renders.
