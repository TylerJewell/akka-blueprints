# User journeys — codeact-sandboxed-agent

Authoritative acceptance criteria. Each journey maps to one or more integration test scenarios.

---

## J1 — Successful task solved in one iteration

**Preconditions:**
- Service is running at `localhost:9337`.
- Model provider is configured (real LLM or mock LLM with option-a).
- Seeded task "word-frequency" is available in `seed-tasks.jsonl`.

**Steps:**
1. `POST /api/tasks` with the word-frequency seed task body (description, contextData containing a 10-word sentence, acceptanceCriterion stating the expected JSON output, submittedBy = "test-user").
2. Receive `201 { "taskId": "t-..." }`.
3. `GET /api/tasks/{taskId}` — poll until status is not `SUBMITTED`.
4. Within 30 s, status transitions to `EXECUTING`.
5. Within 60 s of `EXECUTING`, status transitions to `SOLVED`.
6. `GET /api/tasks/{taskId}` returns `resolution.status = SOLVED`, `resolution.iterationsUsed` in the range 1–3, and `resolution.finalOutput` containing a JSON object where `"the"` maps to `3` and `"fox"` maps to `2`.
7. `outputHistory` has one entry per iteration; `secretCategoriesFound` is `[]` for all entries.
8. The App UI tab shows the task card with a green `SOLVED` badge and the final output in the resolution section.

**Expected:** task moves to `SOLVED` with the correct output within 30 s of submission.

---

## J2 — Guardrail rejects forbidden import; agent rewrites and solves

**Preconditions:**
- Mock LLM is active (option-a) so the first iteration is deterministic.
- The mock response for the 3rd task (modulo seed) selects an entry containing `import subprocess`.

**Steps:**
1. Submit enough tasks (or one task seeded to hit the guardrail-triggering mock entry) to land on a submission where the first mock code contains `import subprocess`.
2. `GET /api/tasks/{taskId}` via SSE.
3. Observe the first `EXECUTING` event: `codeHistory[0].code` contains `import subprocess`.
4. The entity does NOT transition to `HALTED` or `FAILED` after the first iteration — the task remains `EXECUTING`.
5. The next SSE event shows `codeHistory[1].code` without `import subprocess`; the code executes cleanly.
6. Within 30 s of the second iteration, the task transitions to `SOLVED`.

**Expected:** the guardrail rejects the forbidden import silently (from the user's perspective), the agent rewrites, and the second iteration solves the task. The UI never shows an error card — only a two-iteration SOLVED card.

---

## J3 — Safety halt fires on secret-bearing output; task transitions to HALTED

**Preconditions:**
- Mock LLM is active (option-a) and configured to produce an output containing `GITHUB_TOKEN=ghp_AbCdEfGhIjKlMnOpQrStUv012345` for a specific task seed.

**Steps:**
1. Submit a task that triggers the halting mock response.
2. `GET /api/tasks/sse` and observe events.
3. The `EXECUTING` event fires.
4. Within 10 s, the task transitions to `HALTED`.
5. `GET /api/tasks/{taskId}` returns `status = HALTED`, `resolution = null`, `outputHistory = []`.
6. The App UI shows the task card with an orange `HALTED` badge and the halt reason ("matched github-token pattern") in the execution log.
7. The raw output containing the token is NOT accessible via `GET /api/tasks/{taskId}` — it was never written to the entity.

**Expected:** the safety halt catches the secret-bearing output before the `CodeExecuted` event is written; no secret appears in the entity log or the UI.

---

## J4 — Secret sanitizer scrubs credential from output; task solves with redacted form in UI

**Preconditions:**
- Mock LLM produces an output that contains `AWS_SECRET_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE` alongside the correct computation result.
- The string does not trigger the safety halt (no destructive action marker, not a PEM header).

**Steps:**
1. Submit a task with context data that includes a config block containing an AWS key string.
2. The sandbox runs the code; the raw output contains both the correct answer and `AWS_SECRET_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE`.
3. `SafetyHaltMonitor` scans the output — the string matches the AKIA prefix pattern. This triggers a halt.

*(If the deployer has tuned the halt monitor to only flag PEM headers and destructive markers — not AWS keys — then the flow continues:)*

3. `SafetyHaltMonitor` returns `Optional.empty()` (AKIA pattern not in halt ruleset for this configuration).
4. `CodeExecuted` event is written with the raw output.
5. `SecretSanitizer` Consumer fires; replaces `AKIAIOSFODNN7EXAMPLE` with `[REDACTED-AWS-KEY]`; writes `OutputSanitized`.
6. `GET /api/tasks/{taskId}` returns `outputHistory[0].sanitizedOutput` containing `[REDACTED-AWS-KEY]` — not the raw key.
7. `outputHistory[0].secretCategoriesFound = ["aws-key"]`.
8. The App UI shows `[REDACTED-AWS-KEY]` with an amber highlight in the execution log accordion.

**Expected:** the credential appears nowhere in the UI or the view row; the entity retains the raw form for audit-only access.

---

## J5 — Iteration budget exhausted; task transitions to FAILED

**Preconditions:**
- Mock LLM always returns code that produces incorrect output for a specific task seed (all 5 entries in the mock return wrong answers).

**Steps:**
1. Submit the task that triggers the all-wrong mock responses.
2. Observe the task cycling through 5 `EXECUTING` events via SSE.
3. After the 5th iteration, `AcceptanceChecker` returns `false` and the budget is exhausted.
4. `ExecutionWorkflow` calls `TaskEntity.markFailed("iteration budget exhausted")`.
5. `GET /api/tasks/{taskId}` returns `status = FAILED`, `codeHistory` has 5 entries, `outputHistory` has 5 entries, `resolution = null`.
6. The App UI shows a red `FAILED` card with the iteration count chip showing `5 iter`.

**Expected:** the workflow respects the 5-iteration cap and transitions to `FAILED` without hanging or retrying indefinitely.

---

## J6 — SSE client receives real-time updates during execution

**Preconditions:**
- Service is running.
- A long-running task (3-iteration solve in mock LLM) is submitted.

**Steps:**
1. Open an SSE connection to `GET /api/tasks/sse`.
2. Submit the task via `POST /api/tasks`.
3. Observe the following events arrive in order over the SSE stream:
   - `task-update` with `status: "SUBMITTED"` — arrives within 100 ms of submission.
   - `task-update` with `status: "EXECUTING"` and `codeHistory` length 1 — arrives within 5 s.
   - `task-update` with `status: "EXECUTING"` and `codeHistory` length 2 — arrives within 30 s.
   - `task-update` with `status: "SOLVED"` and `resolution.iterationsUsed: 3` — arrives within 60 s.
4. All four events arrive before `GET /api/tasks/{taskId}` returns the final row.

**Expected:** the SSE stream delivers one event per state transition; no polling is needed; a late-joining client receives the full row at reconnect.
