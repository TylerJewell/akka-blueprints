# User journeys

Numbered acceptance journeys. Passing all three means the blueprint generated correctly.

## J1 — Ask via the UI and watch it answer

- **Preconditions:** service running on `http://localhost:9390`; a model provider configured (real key or mock).
- **Steps:**
  1. Open the App UI tab.
  2. Type "What is the capital of France?" and click **Ask**.
- **Expected state:** a `QuestionEntity` is created; status `ASKED`, then `ANSWERED` after the agent completes (≈5–30s with a real model, immediate with mock).
- **Expected UI:** the question appears in the list, shows "thinking…", then shows the answer text and a confidence percentage with a green `ANSWERED` pill.
- **Done when:** the row shows non-empty answer text and a confidence in `[0,1]`.

## J2 — Ask via the API and read the typed answer

- **Preconditions:** service running.
- **Steps:**
  1. `POST /api/ask` with `{ "question": "What is 2 + 2?" }`; capture `questionId`.
  2. Poll `GET /api/questions/{questionId}` until `status` is `ANSWERED`.
- **Expected state:** `AnswerRecorded` event applied; `answerText` and `confidence` populated.
- **Expected response:** the `Question` JSON includes the answer text and a numeric `confidence`.
- **Done when:** the typed answer round-trips through the API.

## J3 — A low-quality answer is blocked

- **Preconditions:** service running. With the mock provider, the seeded responses include at least one entry with `confidence < 0.4`.
- **Steps:**
  1. Ask a question whose answer comes back below the threshold (with the mock provider, ask repeatedly until the low-confidence entry is returned; with a real model, ask an unanswerable question such as "What number am I thinking of?").
- **Expected state:** the before-agent-response check fails; `AnswerConsumer` writes `AnswerBlocked`; status `BLOCKED`.
- **Expected UI:** the row shows a red `BLOCKED` pill and a short reason, with no answer text.
- **Done when:** a question reaches `BLOCKED` and no answer text is shown.

## J4 — Streaming list stays current

- **Preconditions:** service running; App UI tab open with the SSE stream connected.
- **Steps:**
  1. Ask two questions in quick succession.
- **Expected UI:** both rows appear and update through `ASKED` → terminal state without a manual refresh.
- **Done when:** the list reflects every status change live over `/api/questions/sse`.
