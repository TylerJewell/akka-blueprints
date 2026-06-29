# User journeys — akka-resumable-agent-http

## J1 — Start a run and receive a weather report

**Preconditions:** Service running on declared port (`http://localhost:9940/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9940/` → App UI tab.
2. From the **Query type** dropdown, pick `CURRENT`.
3. In the **Location** field, type `San Francisco, CA` (or click **Use seeded example**).
4. Click **Run agent**.

**Expected:**
- The new card appears in the live list with status `INITIATED` within 1 s.
- Within 1 s, the card transitions to `TOOL_CALLED`. The animated progress indicator is visible; the status pill is yellow.
- Within 30 s (or less with mock LLM), the card reaches `COMPLETED`. The right pane shows: the `WeatherReport` with a non-empty `summary`, a numeric `temperatureCelsius`, and a `conditions` string. `resumeCount` is `0`. No incident panel appears.

## J2 — Kill the process during tool execution; verify checkpoint resume

**Preconditions:** Service running with `slow-weather-tool.delay-seconds = 8` (default). Any model provider.

**Steps:**
1. Start a run (J1 steps 1–4). Wait for the status to show `TOOL_CALLED`.
2. Kill the process (Ctrl-C in the terminal running `/akka:build`, or send SIGKILL).
3. Restart the service via `/akka:build`.
4. Open the App UI and navigate to the same run by `runId`.

**Expected:**
- The run's status transitions to `RESUMED` within 2 s of restart.
- The `resumeCount` on the card increments to 1. The resume badge `Resumed ×1` appears.
- The workflow resumes `toolCallStep` (if the tool had not yet returned) or `reportStep` (if it had). The `initStep` is NOT re-executed — the `RunInitiated` event timestamp from before the crash is unchanged on the entity.
- Within 30 s of restart, the run reaches `COMPLETED` with the same `runId` and a `WeatherReport`.
- The incident panel appears in the right pane with `severity = WARNING`, `interruptedStep = toolCallStep`, a positive `elapsedBeforeCrashMs`, and a notes string describing the resume.

## J3 — before-tool-call guardrail blocks an invalid location

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `query-weather.json` includes an entry with an empty location argument.

**Steps:**
1. Observe a run whose mock seed produces a malformed tool invocation (the mock selects a bad entry on the first iteration of every 4th run — modulo seed).
2. Watch the run's card in the UI and in the service log.

**Expected:**
- The first tool invocation carries an empty `location` argument.
- `ToolCallGuardrail` rejects the invocation. The service log shows one `guardrail.reject` line with check name `location-empty`.
- The malformed invocation NEVER reaches `SlowWeatherTool.execute(...)` — no 8-second delay for the rejected call.
- The agent loop retries on iteration 2 with a non-empty location. The retry invocation passes the guardrail and proceeds normally.
- The run card shows the red **Guardrail** chip with `rejectionReason = location-empty`. The card still completes successfully with `status = COMPLETED`.

## J4 — Incident severity is WARNING when tool call was in-flight at crash

**Preconditions:** Same as J2. The process is killed after `ToolCallStarted` is emitted but before `ToolCallCompleted`.

**Steps:**
1. Start a run. Wait for the `TOOL_CALLED` status (card turns yellow).
2. Kill the process immediately (within the 8-second tool delay).
3. Restart and wait for `COMPLETED`.

**Expected:**
- The incident panel shows `severity = WARNING` (not `INFO` and not `ERROR`).
- `interruptedStep = toolCallStep`.
- `elapsedBeforeCrashMs` is between 0 and `slow-weather-tool.delay-seconds * 1000` ms, confirming the crash happened during the tool call.

## J5 — Incident severity is INFO when crash happens before tool call

**Preconditions:** Service running. The process is killed between `RunInitiated` and `ToolCallStarted`.

**Steps:**
1. Start a run. Kill the process immediately after the `INITIATED` status appears (before the yellow `TOOL_CALLED` transition).
2. Restart and wait for `COMPLETED`.

**Expected:**
- The incident panel shows `severity = INFO`.
- `interruptedStep = initStep`.
- The `WeatherReport` lands on first tool attempt after resume; no guardrail chip appears if the tool arguments were valid.
