# User journeys — sandboxed-code-agent

Four acceptance journeys covering the key paths through the system. Each journey has preconditions, numbered steps, and expected outcomes. The journeys are numbered to match the acceptance tests cited in `SPEC.md §10`.

---

## J1 — Happy path: data-analysis task completes end-to-end

**Preconditions:**
- Service is running (local Docker daemon is available).
- `SANDBOX_BACKEND` is `DOCKER` (default).
- Mock LLM is active or a valid model-provider key is configured.

**Steps:**
1. Open the App UI tab. Click **Load seeded task** and select the data-analysis seed ("Compute mean and std dev of [1, 4, 9, 16, 25]"). Wall-clock budget: 30 s. CPU budget: 10 s.
2. Click **Run task**.
3. Observe the live list: a new card appears with status `SUBMITTED`, then transitions to `SCREENING` within ~1 s.
4. Observe the card transition to `APPROVED` (guardrail passed), then `RUNNING`.
5. Within 30 s, the card transitions to `COMPLETED`.
6. Select the card. In the right panel: the generated Python code is visible; the screening outcome shows `APPROVED`; stdout shows `Mean: 11.00\nStd dev: 8.35`; exit code badge shows `0`.

**Expected:**
- The execution reaches `COMPLETED` within the wall-clock budget.
- `stdout` contains the computed statistics.
- No BLOCKED chip is visible on this card.
- The `ExecutionView` row for this execution has `status = COMPLETED` and non-null `output`.

---

## J2 — Guardrail blocks a forbidden pattern; agent revises and succeeds

**Preconditions:**
- Mock LLM is active (or the model-provider is set to a model that can be prompted to produce a forbidden pattern on first attempt).
- The mock is seeded so the first iteration of every 3rd execution uses a response that contains `os.environ.get("ANTHROPIC_API_KEY")` in the generated code.

**Steps:**
1. Submit three executions using the data-analysis seed task.
2. On the third submission, observe the live list: the card enters `SCREENING`.
3. Observe a `BLOCKED: env-harvest` chip appear on the card (the guardrail rejected the first tool call).
4. Within the agent's remaining iterations, observe the card's screening chip change to `APPROVED` and the card transition to `RUNNING`, then `COMPLETED`.
5. In the right panel, the generated code block shows the approved Python (no `os.environ` call). The BLOCKED chip for the prior attempt is displayed above the approved code with a muted style.

**Expected:**
- The entity log contains a `CodeBlocked` event for the first attempt and a `CodeApproved` event for the second.
- The UI never shows the blocked code as the active code — it is shown only as a historical note.
- The execution completes normally; the operator can see exactly which pattern was blocked.

---

## J3 — Runaway process hits wall-clock budget; safety halt fires

**Preconditions:**
- Service is running with Docker back-end.
- Wall-clock budget is set to 5 s for this execution.

**Steps:**
1. In the App UI, type the task description: "Run an infinite loop that counts forever." Set wall-clock budget to 5 s, CPU budget to 10 s.
2. Click **Run task**.
3. The card enters `SCREENING`, then `APPROVED` (the code `while True: pass` contains no forbidden patterns), then `RUNNING`.
4. After ~5 s, observe the card transition to `HALTED`.
5. Select the card. The halt section shows: `limit_breached: wall-clock`, `measured: ~5010 ms`, `budget: 5000 ms`.

**Expected:**
- The entity emits `ExecutionHalted` with a `HaltReason` identifying `wall-clock` as the breached limit.
- The Docker container is no longer running (verify with `docker ps` if needed).
- The `ExecutionView` row has `status = HALTED` and non-null `haltReason`.
- The card border in the UI is highlighted in orange.

---

## J4 — Host secrets do not appear in the LLM call log

**Preconditions:**
- A real model-provider key (`ANTHROPIC_API_KEY`) is set in the environment.
- Request-level logging is enabled (set `akka.javasdk.agent.log-requests = true` in `application.conf` for this test, or check the Akka local console's agent-call log).

**Steps:**
1. Submit any task via the App UI.
2. After the execution completes, inspect the agent's call log in the Akka local console (port 9889) or the service's stdout.
3. Search the log for the value of `ANTHROPIC_API_KEY` (or the first 8 chars of it).
4. Search the log for `E2B_API_KEY` (if that env var is set).

**Expected:**
- The LLM call log contains only the task description text and the generated Python code.
- No environment variable values appear anywhere in the model's input or output as logged.
- The `execute_code` tool parameter in the log shows only `code: "..."` — no env or credential strings.

---

## J5 — File-transformation task produces expected filtered output

**Preconditions:**
- Service is running with Docker back-end.
- Mock LLM or a configured model is active.

**Steps:**
1. Click **Load seeded task** and select the file-transformation seed ("Filter CSV rows where score > 60 and print the names").
2. Click **Run task** with default budgets.
3. Observe the card reach `COMPLETED`.
4. In the right panel, stdout shows the two names from the CSV whose score exceeds 60 (Alice: 85, Carol: 91).

**Expected:**
- Exit code is 0.
- stdout contains exactly two lines: `Alice: 85` and `Carol: 91`.
- `generatedFiles` is empty (this task produces only console output).
- Eval score (if an eval step is wired) would be N/A — this blueprint does not include an on-decision eval; the panel simply shows COMPLETED with the output.
