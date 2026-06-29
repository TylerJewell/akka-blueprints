# User journeys — akka-basic

## J1 — Submit a summarisation prompt and get a result

**Preconditions:** Service running on declared port (`http://localhost:9383/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9383/` → App UI tab.
2. From the **Prompt category** dropdown, pick `Summarise`.
3. Click **Load seeded example** to fill the prompt textarea with the seeded summarisation text.
4. Click **Run prompt**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `PROCESSING` within 1 s.
- Within 30 s the card reaches `COMPLETED`. The right pane shows: a category badge `SUMMARISE`, a confidence bar at or above 60 %, and a non-empty `outputText` (1–3 sentences condensing the seeded text).

## J2 — Guardrail blocks a structurally invalid result

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `process-prompt.json` includes a deliberately malformed entry with an empty `outputText`.

**Steps:**
1. Submit any seeded prompt three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/runs/sse`).

**Expected:**
- The third submission's first agent iteration produces a result with empty `outputText`.
- The `after-agent-response` guardrail rejects it. The malformed result NEVER lands in `PromptEntity` — there is no `ResultRecorded` event with an empty payload.
- The agent loop retries on iteration 2 and produces a valid result. The card transitions to `COMPLETED` with a non-empty `outputText`.
- The service log shows one `guardrail.reject` line for the rejected iteration, naming `outputText` as the failing check.

## J3 — Out-of-range confidence score is caught

**Preconditions:** Mock LLM mode. The mock's `process-prompt.json` includes a malformed entry with `confidenceScore = 1.5`.

**Steps:**
1. Submit any seeded prompt such that the mock selects the out-of-range entry on the first iteration (the mock uses a deterministic seed based on `runId` modulo 3 — see `MockModelProvider.seedFor`).
2. Wait for `COMPLETED`.

**Expected:**
- The first agent iteration produces `confidenceScore = 1.5`.
- The guardrail rejects it with a rejection message naming `confidenceScore` and the actual value.
- The second iteration produces a valid result with `confidenceScore` in `[0.0, 1.0]`. The card reaches `COMPLETED`.
- The entity log never contains a `ResultRecorded` event with `confidenceScore = 1.5`.

## J4 — Concurrent runs complete independently

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the summarisation seed prompt.
2. Without waiting for COMPLETED, immediately submit the extraction seed prompt.
3. Wait for both cards to reach `COMPLETED`.

**Expected:**
- Both cards reach `COMPLETED` independently. Neither blocks the other.
- The `outputText` of the summarisation run reflects summarisation behavior; the `outputText` of the extraction run reflects extraction behavior.
- Fetching `GET /api/runs/{id1}` and `GET /api/runs/{id2}` returns two distinct `Run` records with no shared state.

## J5 — Classification hint is respected

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the classification seed prompt with `categoryHint = CLASSIFY`.
2. Wait for `COMPLETED`.

**Expected:**
- The result's `category` field is `CLASSIFY`.
- The `outputText` begins with a label word followed by a one-sentence justification.
- The `confidenceScore` is in `[0.0, 1.0]`.

## J6 — AUTO hint resolves to a category

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the summarisation seed prompt with `categoryHint = AUTO`.
2. Wait for `COMPLETED`.

**Expected:**
- The result's `category` field is one of `{SUMMARISE, EXTRACT, CLASSIFY}` — not `AUTO` (AUTO is a hint; the agent resolves it to a concrete category).
- The `outputText` is consistent with the resolved category's behavior.
