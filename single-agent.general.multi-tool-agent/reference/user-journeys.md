# User journeys — multi-tool-agent

## J1 — Submit a multi-part request and get a combined answer

**Preconditions:** Service running on declared port (`http://localhost:9172/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9172/` → App UI tab.
2. Click the seeded example chip: "What is 72°F in Celsius, and how many euros is $85?" — this fills the Request textarea.
3. Click **Submit**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `DISPATCHING` within 1 s.
- The tool-call log in the right pane shows `UnitTool` and `CurrencyTool` entries appearing as each tool call completes. Both entries have non-null `toolName`, `inputArgs`, `result`, and `calledAt`. `guardRejectionReason` is null for both.
- Within 30 s, the card reaches `ANSWERED`. The answer paragraph appears: a sentence about the Celsius conversion and a sentence about the euro amount.
- `finishedAt` is set; the session is terminal.

## J2 — Guardrail rejects an invalid currency code and the agent recovers

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-request.json` includes entries whose first CurrencyTool call uses an invalid `fromCode` (`EURO` instead of `EUR`).

**Steps:**
1. Submit any seeded request that includes a currency conversion. Repeat until the third submission (the mock selects a rejection-containing entry every 3rd session by seed).
2. Watch the tool-call log in the right pane as events arrive.

**Expected:**
- The first `CurrencyTool` entry in the log shows `guardRejectionReason: "fromCode 'EURO' is not a valid ISO-4217 code"` and `result` set to the guardrail error text.
- A second `CurrencyTool` entry follows with `fromCode: "EUR"` and `guardRejectionReason: null`.
- The session reaches `ANSWERED` with a valid combined answer.
- The service log shows one `guardrail.reject` line naming `CurrencyTool`, `fromCode`, and the rejection reason.

## J3 — Iteration budget exhausted lands the session in FAILED

**Preconditions:** Mock LLM mode. The mock is configured to emit a `ToolCallRecord` entry with a guardrail rejection on every iteration, exhausting all 5.

**Steps:**
1. Send a request text known to trigger the exhaustion mock path (see `src/main/resources/mock-responses/answer-request.json` "exhaust" entry).
2. Wait for the session to finish.

**Expected:**
- The card status reaches `FAILED`.
- The right pane shows a red banner with the failure reason (e.g., "Agent exhausted iteration budget without producing a ToolResponse.").
- The partial tool-call log is visible — all 5 rejected attempts are listed with their `guardRejectionReason`.
- No `SessionAnswered` event appears in the SSE stream; only `SessionFailed`.

## J4 — Every tool-call log entry is complete

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded example "Tell me the weather in London and convert 500 GBP to USD."
2. Wait for `ANSWERED`.
3. Fetch `GET /api/sessions/{id}` and read the `toolCalls` array.

**Expected:**
- The `toolCalls` array has 2 entries: one for `WeatherTool` and one for `CurrencyTool`.
- Each entry has non-null: `callId`, `toolName`, `inputArgs` (non-empty map), `result` (non-empty string), `calledAt`. `guardRejectionReason` is null for both.
- The UI's tool-call log table shows both rows with all columns filled.

## J5 — Weather look-up for an unknown city returns a graceful answer

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Type a custom request: "What is the weather in Atlantis?" (not in the seeded city map).
2. Submit.

**Expected:**
- `WeatherTool` is called with `cityName: "Atlantis"`.
- The guardrail accepts the call (city name is non-empty and under 100 characters).
- The tool returns a "not found" sentinel result.
- The agent's answer paragraph includes a clear statement that weather data for Atlantis is not available in this environment.
- The session reaches `ANSWERED`, not `FAILED`.
