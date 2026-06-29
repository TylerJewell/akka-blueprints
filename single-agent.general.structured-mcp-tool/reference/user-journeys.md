# User journeys — structured-mcp-tool

## J1 — Submit a location and get a weather summary

**Preconditions:** Service running on declared port (`http://localhost:9215/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9215/` → App UI tab.
2. Click the **London** quick-pick button. The Location input fills with "London". Leave unit at Celsius.
3. Enter any value in **Submitted by**.
4. Click **Get weather**.

**Expected:**
- A new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `RUNNING` within 1 s as the workflow starts.
- Within 30 s the card reaches `COMPLETED`. The right pane shows: a conditions badge, a temperature chip, a humidity chip, a wind speed chip, and a one-sentence narrative paragraph.
- All numeric fields are populated (no nulls visible in the UI).
- The `finishedAt` timestamp is present on `GET /api/queries/{id}`.

## J2 — Guardrail rejects a payload missing a required field

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `InlineMcpServer` is configured to return a payload missing `humidityPercent` on the first tool call of every 3rd query (modulo seed).

**Steps:**
1. Submit three queries in succession: London, Tokyo, Sydney.
2. Watch the third query's lifecycle in the network panel (`/api/queries/sse`).

**Expected:**
- The third query's first tool-call iteration returns a payload without `humidityPercent`.
- `McpToolResultGuardrail` rejects it. The malformed payload NEVER appears in the result pane — the `SummaryRecorded` event is not emitted with an incomplete payload.
- The agent retries `get_weather_info` on iteration 2. The retry returns a valid payload; the card transitions to `COMPLETED` with all fields populated.
- The service log shows one `guardrail.reject` line with code `missing-required-field` for the rejected iteration.

## J3 — Guardrail rejects a temperature out of physical range

**Preconditions:** Service running with the mock LLM selected. The mock's `InlineMcpServer` includes an entry with `temperatureCelsius = 9999`.

**Steps:**
1. Submit a query for a location whose mock seed maps to the out-of-range entry (see `src/main/resources/mock-responses/get-weather-summary.json`).
2. Check the service log and the SSE stream.

**Expected:**
- The first tool-call iteration returns `temperatureCelsius: 9999`.
- The guardrail rejects with code `temperature-out-of-range`. The query does not immediately fail.
- The agent retries; the retry returns a valid payload; the card reaches `COMPLETED`.
- The service log shows exactly one `guardrail.reject` line with `temperature-out-of-range` before the successful iteration.

## J4 — Late-joining SSE client sees current state

**Preconditions:** Service running. At least one query in `COMPLETED` state.

**Steps:**
1. Open `GET /api/queries/sse` in a new browser tab (or via `curl`) after the query has already reached `COMPLETED`.
2. Observe the first event delivered.

**Expected:**
- The stream immediately delivers the full current row of every query that has already transitioned, including the `COMPLETED` query with its full `summary` object.
- The client does not need to call `GET /api/queries/{id}` to get the current state — the SSE event carries the complete row.
- No duplicate events are delivered; the event for the already-completed query appears exactly once on connect.

## J5 — Fahrenheit unit preference

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Click the **Sydney** quick-pick button.
2. Toggle the unit selector to **Fahrenheit**.
3. Click **Get weather**.

**Expected:**
- The result pane shows both temperature values: `temperatureFahrenheit` and `temperatureCelsius`.
- The narrative paragraph uses the Fahrenheit value (e.g., "Sydney is clear at 77.0 °F with light winds.").
- `GET /api/queries/{id}` returns a `summary` object where `temperatureFahrenheit` is non-null and the value equals `(temperatureCelsius × 9/5) + 32` within 0.1 °F.
