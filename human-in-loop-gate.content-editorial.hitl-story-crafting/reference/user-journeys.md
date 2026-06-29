# User journeys

Acceptance journeys the generated system must pass.

## J1 — Start a story

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/stories` with `{ "premise": "A lighthouse keeper discovers a sealed door that was never in the original blueprints." }`.
- **Expected:** response returns `{ storyId }`. Within ~30 s the story appears via SSE in `AWAITING_DIRECTION` with `turnCount: 1` and one chapter entry with non-empty `title` and `body`. The App UI card shows Continue and End buttons.

## J2 — Continue the story with direction

- **Preconditions:** a story in `AWAITING_DIRECTION` with one chapter (J1).
- **Steps:** `POST /api/stories/{storyId}/continue` with `{ "direction": "The keeper opens the door and finds a room full of antique radio equipment still transmitting." }`.
- **Expected:** story transitions to `DRAFTING`; within ~30 s status returns to `AWAITING_DIRECTION` with `turnCount: 2` and two chapter entries. The second chapter incorporates the radio-room direction. Continue and End buttons remain visible.

## J3 — End the story

- **Preconditions:** a story in `AWAITING_DIRECTION` (J1 or J2).
- **Steps:** `POST /api/stories/{storyId}/end`.
- **Expected:** story transitions to terminal `COMPLETED`; `completedAt` is non-null; no further chapters are written; Continue and End buttons disappear from the UI card; the full chapter list remains visible.

## J4 — Content guard blocks a draft

- **Preconditions:** mock provider configured with a content-guard-agent.json entry where `passed: false`.
- **Steps:** `POST /api/stories` with any premise that routes to the failing mock entry.
- **Expected:** `StoryWorkflow` calls `ContentGuardAgent`; receives `GuardResult{passed: false, reason: "..."}` ; calls `storyScreenedOut` on `StoryEntity`; story transitions to terminal `SCREENED_OUT` with a non-null `screenOutReason`; no chapter is persisted (`chapters` list is empty); the UI card shows the guard reason in muted red.

## J5 — No chapter without direction

- **Preconditions:** a story in `AWAITING_DIRECTION` (J1).
- **Steps:** wait 15 seconds without calling continue or end.
- **Expected:** story status remains `AWAITING_DIRECTION`; `turnCount` does not increase; `StoryWorkflow` is polling on its 5-second timer but produces no new chapter. The workflow does not time out or error during the wait.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 2 controls (H1, G1) in `matrix-row` style with mechanism pills (`hitl` yellow pill on H1, `guardrail` red pill on G1); Risk Survey renders pre-filled answers with `TO_BE_COMPLETED_BY_DEPLOYER` values shown muted italic; the Overview tab renders the README content.
