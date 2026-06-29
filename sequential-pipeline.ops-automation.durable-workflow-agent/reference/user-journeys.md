# User journeys — durable-workflow-backed-agent

## J1 — Submit an incident and watch it resolve

**Preconditions:** Service running on `http://localhost:9731/`; a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded incident `high-latency-api-gateway` has a matching `src/main/resources/sample-data/incidents/api-gateway.json` file.

**Steps:**
1. Open `http://localhost:9731/` → App UI tab.
2. From the **Pick a seeded incident** dropdown, pick `high-latency-api-gateway`.
3. Click **Remediate**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `DIAGNOSING` within 1 s more.
- Within ~20 s the card reaches `DIAGNOSED`. The right pane shows the Diagnosis panel: root-cause text, severity badge `critical`, evidence-metrics table with ≥ 2 rows, evidence-logs table with ≥ 1 row.
- Within ~20 s more the card reaches `REMEDIATED`. The right pane shows ≥ 1 action in the actions-applied list, each with `outcome: applied`.
- Within ~20 s more the card reaches `VERIFIED`. The right pane shows ≥ 1 health check with `healthy: true` and a resolution-check row with `resolved: true`.
- `budgetViolations` is an empty array on this happy path; no red dot appears on the card.
- Total elapsed time: ≤ 60 s.

## J2 — Budget-cap guardrail fires and workflow recovers

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `diagnose-incident.json` includes one entry whose `tool_calls` array has 9 calls (exceeding the DIAGNOSE cap of 8). This entry is selected on the FIRST iteration of every 4th incident (modulo seed).

**Steps:**
1. Submit any seeded incident four times in a row (J1 steps × 4).
2. Watch the fourth submission's lifecycle in the network panel of the browser dev tools (`/api/incidents/sse`).

**Expected:**
- On the fourth submission's `diagnoseStep`, the agent's 9th tool call is `fetchLogs`. `BudgetGuardrail` halts it; a `BudgetExhausted{phase: "DIAGNOSE", tool: "fetchLogs", callsUsed: 8, cap: 8}` event lands on the entity.
- The halted call NEVER reaches `DiagnoseTools.fetchLogs` — there is no log line from the tool body for that call.
- The workflow's step recovery retries `diagnoseStep` once. The retry uses a fresh agent task; the mock's seed selects a normal (non-violating) entry. The incident eventually reaches `VERIFIED`.
- The card in the App UI shows the small red dot indicating a budget violation fired. The budget-violation strip on the right pane shows one row: phase `DIAGNOSE`, tool `fetchLogs`, callsUsed/cap `8/8`, violatedAt timestamp.

## J3 — Scheduled durability evaluator computes metrics

**Preconditions:** Service running. At least 3 incidents have been submitted and closed (reached `VERIFIED` or `FAILED`). At least one had a budget violation (J2 run recommended first).

**Steps:**
1. Click the **Evaluate now** button on the Overview tab (which posts to `POST /api/durability/trigger`).
2. Wait for the four-metric tile row on the Overview tab to populate.

**Expected:**
- The `GET /api/durability/latest` response is non-null.
- `incidentsEvaluated` reflects the number of incidents closed since the last evaluation window started (or all incidents if this is the first trigger).
- `completionRatePercent` is ≥ 66 if at least 2 of 3 incidents reached `VERIFIED`.
- `budgetViolationRatePercent` is > 0 if the J2 path ran.
- `mttrMinutes` is a positive float representing the mean time from `CREATED` to `VERIFIED` (or `FAILED`) across the evaluated incidents.
- The four metric tiles on the Overview tab render the values from the report.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded incident.
2. Wait for `VERIFIED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the incidentId.

**Expected:**
- The DIAGNOSE task's log entries show only `queryMetrics` and `fetchLogs` calls.
- The REMEDIATE task's log entries show only `applyAction` and `confirmAction` calls.
- The VERIFY task's log entries show only `checkHealth` and `confirmResolved` calls.
- No cross-phase calls appear.
- The order of tasks in the log is DIAGNOSE → REMEDIATE → VERIFY. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Step timeout triggers workflow failover

**Preconditions:** Mock LLM mode. One of the mock entries is configured to produce a 95-second delay in `diagnoseStep` (or the step timeout is temporarily lowered to 5 s for testing purposes via a test config override).

**Steps:**
1. Submit an incident whose mock seed selects the delayed entry.
2. Wait for the incident to reach `FAILED`.

**Expected:**
- The step timeout fires before the agent returns from `diagnoseStep`.
- The workflow's default step recovery exhausts its `maxRetries(2)` budget (if the delay persists on retry) and invokes the error step.
- The error step writes `IncidentFailed{reason: "step-timeout: diagnoseStep"}` onto the entity.
- The incident card shows status `FAILED` with a red border.
- `DurabilityEvaluator` on next trigger counts this incident under `timeoutRatePercent`.

## J6 — Custom incident with no matching sample data

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/incidents/<custom-serviceId>.json` exists.

**Steps:**
1. In the App UI, enter a custom `alertId` (e.g., `mystery-alert`) and a custom `serviceId` (e.g., `unknown-service`) into the fields.
2. Click **Remediate**.

**Expected:**
- `DiagnoseTools.queryMetrics` returns an empty list; `DiagnoseTools.fetchLogs` returns an empty list.
- The agent's DIAGNOSE task returns a `DiagnosisReport` with `rootCause = "(no evidence available)"`, `severity = "medium"`, and empty evidence lists.
- The workflow advances to `remediateStep`. The agent's REMEDIATE task applies a generic action and returns a `RemediationPlan` with `outcome = "partial"`.
- The workflow advances to `verifyStep`. The agent returns a `VerificationResult` with `resolved = false` and a `resolutionCheck.evidence` noting insufficient data.
- The incident reaches `VERIFIED` (the workflow completes — `VERIFIED` means the verification phase completed, not that the incident was necessarily confirmed fixed). Nothing crashes; the partial result is honestly surfaced.
