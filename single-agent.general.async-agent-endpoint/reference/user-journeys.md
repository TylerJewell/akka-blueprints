# User journeys — async-agent-endpoint

## J1 — Submit a seeded task and get a result

**Preconditions:** Service running on declared port (`http://localhost:9526/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9526/` → App UI tab.
2. From the **Task** dropdown, pick `Vowel counter`.
3. Confirm the prompt textarea reads: `"Count the number of vowels in the string 'supercalifragilistic'"`.
4. Click **Run task**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `RUNNING` within 1 s.
- Within 30 s the card reaches `COMPLETED`. The right pane shows: the generated Python code in a monospace block, an empty stdout block (the result is in outputValue, not printed), and an `outputValue` badge showing `9`.
- The guardrail-hit badge is absent (no policy violation on the happy path).

## J2 — Guardrail blocks a policy-violating tool call

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `run-task.json` includes entries with `subprocess` import on the first iteration for every 3rd run.

**Steps:**
1. Submit any seeded task three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel (`/api/runs/sse`).

**Expected:**
- The third submission's first agent iteration generates code containing `import subprocess`.
- The `before-tool-call` guardrail rejects it. The forbidden code NEVER executes — there is no tool-call result for the `subprocess` invocation.
- The agent loop retries on iteration 2 and produces compliant code. The card reaches `COMPLETED` with the expected `outputValue`.
- The guardrail-hit badge is shown (amber) on the completed card.
- The service log shows one `guardrail.reject` line for the first iteration with `construct: "subprocess"`.

## J3 — Execution error is recorded as FAILED

**Preconditions:** Mock LLM mode. A specific run seed causes the mock to return code that raises a `ZeroDivisionError`.

**Steps:**
1. Submit a custom prompt: `"Divide 10 by 0 and return the result"`.
2. Wait for the run to settle.

**Expected:**
- The card reaches `FAILED` with `FailureKind.EXECUTION_ERROR`.
- The right pane shows the `stderr` content containing the Python traceback (`ZeroDivisionError: division by zero`).
- The `outputValue` is empty.
- No guardrail-hit badge is shown (the code was syntactically fine; the guardrail passed it through; the error happened at execution time).

## J4 — Concurrent runs do not block each other

**Preconditions:** Service running. Any model provider. Two browser tabs or two `curl` processes available.

**Steps:**
1. In tab A, submit the `Fibonacci` seeded task.
2. Immediately (within 1 s) in tab B, submit the `CSV average` seeded task.
3. Watch both cards in the live SSE stream.

**Expected:**
- Both cards transition to `RUNNING` without one waiting for the other to finish.
- The `createdAt` timestamps of the two `RunStarted` events overlap (both show `RUNNING` before either shows `COMPLETED`).
- Both cards eventually reach `COMPLETED` with correct and independent results.
- The service log does not show one run's step waiting behind the other's at the workflow-dispatch layer.

## J5 — Guardrail exhaustion transitions entity to FAILED

**Preconditions:** Mock LLM mode with a mock entry that returns policy-violating code on all three iterations for a specific run seed.

**Steps:**
1. Submit a task that consistently triggers the all-violating mock entry (consult `mock-responses/run-task.json` for the `guardrail_exhausted` seed value and set `submittedBy` to match).

**Expected:**
- The card reaches `FAILED` with `FailureKind.GUARDRAIL_EXHAUSTED`.
- The `failure.message` names the forbidden construct that was consistently produced (e.g., `"subprocess import"`).
- The service log shows three `guardrail.reject` lines followed by the workflow `error` step transition.
- No generated code was executed on any of the three rejected iterations.
