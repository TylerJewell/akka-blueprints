# User journeys — react-math-wiki

## J1 — Submit a numeric word-problem and get an answer

**Preconditions:** Service running on declared port (`http://localhost:9355/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9355/` → App UI tab.
2. From the **Seeded questions** dropdown, pick `"What is 17% of 850?"`.
3. Click **Ask**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `RUNNING` within 1 s.
- Within 30 s the card reaches `ANSWERED`. The right-pane tool-call trace shows at least one `EVALUATE_MATH` step with a numeric result. The `answerText` contains the numeric result and a unit or description. `confidence` is `HIGH`.
- Within 1 s of `ANSWERED`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — Submit a factual lookup and get a cited answer

**Preconditions:** Service running on declared port.

**Steps:**
1. From the **Seeded questions** dropdown, pick `"What year was the Eiffel Tower completed, and how many years ago was that?"`.
2. Click **Ask**.

**Expected:**
- The tool-call trace in the right pane shows at least one `SEARCH_WIKIPEDIA` step. The observation column contains a passage about the Eiffel Tower.
- If the question requires a calculation (years elapsed), a second `EVALUATE_MATH` step appears in the trace.
- The `answerText` names 1889 and cites the number of years. `confidence` is `HIGH`.
- The eval score is 4 or 5.

## J3 — Guardrail blocks a forbidden math expression; agent reformulates

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-question.json` includes a malformed entry whose trace contains an `evaluate_math` argument with a semicolon.

**Steps:**
1. Submit any seeded question three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel (`/api/questions/sse`) and in the service log.

**Expected:**
- The third submission's first agent iteration emits an `evaluate_math` call with a forbidden construct.
- The `MathGuardrail` fires `before-tool-call` and rejects it. The service log shows one `guardrail.reject` line naming the forbidden token.
- The agent loop counts the rejection as an iteration and retries with a conforming expression.
- The final answer is well-formed and lands in `ANSWERED`. The trace shows the rejected call followed by the corrected one.
- The `MathEvaluator` never receives the forbidden expression.

## J4 — Incomplete answer receives eval score 1

**Preconditions:** Mock LLM mode. The mock contains an entry with a one-word `answerText` and an empty `trace`.

**Steps:**
1. Submit a custom question whose `questionId` maps (via `seedFor`) to the "incomplete" mock entry.
2. Wait for `EVALUATED`.

**Expected:**
- The answer lands well-formed (the guardrail validates tool calls, not the answer structure).
- The eval score chip shows **1** and the rationale reads "Answer is too short and the tool-call trace is empty; no reasoning is visible."
- The card's border highlights red. The user can see the incomplete answer is flagged.

## J5 — Real-time trace visible before answer lands

**Preconditions:** Service running with a real LLM key. A mixed question (factual + math) is submitted.

**Steps:**
1. Submit `"How many meters is the circumference of the Earth, and what is that in kilometers?"`.
2. Watch the right-pane tool-call trace while the question is in `RUNNING`.

**Expected:**
- As soon as the first tool call completes (SEARCH_WIKIPEDIA or EVALUATE_MATH), a row appears in the trace table via SSE — before the answer is ready.
- The trace grows one row per tool call. The user sees the agent's reasoning steps in real time.
- The final answer arrives without a page refresh; the card transitions smoothly from `RUNNING` → `ANSWERED` → `EVALUATED`.
