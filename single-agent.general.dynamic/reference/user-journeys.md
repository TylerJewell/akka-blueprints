# User journeys — Dynamic Route Agent

Acceptance journeys. Passing all of them means the blueprint generated correctly.

## J1 — Summarize text

- **Preconditions:** service running on `http://localhost:9133/`; a model provider key set, or the mock model selected.
- **Steps:**
  1. Open the App UI tab, select route SUMMARIZE, paste a multi-sentence paragraph, submit.
  2. Observe the new request card over SSE.
- **Expected:** `POST /api/requests` returns a `requestId`. The card appears in `SUBMITTED`, then transitions to `COMPLETED` within ~30 s with a non-empty `output` shorter than the input.

## J2 — Translate text

- **Preconditions:** as J1.
- **Steps:**
  1. Select route TRANSLATE, enter text and a target language (e.g. French), submit.
  2. Observe the card.
- **Expected:** the card reaches `COMPLETED` with `output` containing the translated text and `targetLanguage` set.

## J3 — Blocked configuration (G1, before-agent-invocation)

- **Preconditions:** as J1.
- **Steps:**
  1. Submit a request with empty text, or with an unknown route, or TRANSLATE with no target language.
- **Expected:** the request lands in `BLOCKED` with a `blockedReason`; no agent call is made; `output` stays null.

## J4 — Blocked output (G2, before-agent-response)

- **Preconditions:** as J1.
- **Steps:**
  1. Submit a SUMMARIZE request engineered (or mock-configured) so the agent returns the input unchanged or empty output.
- **Expected:** the response guardrail rejects it; the request lands in `BLOCKED` with a `blockedReason`; the bad output never becomes the result.

## J5 — Background simulator

- **Preconditions:** service running; no user interaction.
- **Steps:**
  1. Leave the App UI tab open and wait.
- **Expected:** within ~45 s a simulator-seeded request appears over SSE, runs through the same submit path and guardrails, and reaches a terminal state.
