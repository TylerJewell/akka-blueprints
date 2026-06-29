# User journeys — hook-instrumentation

Acceptance criteria for the generated system. Each journey defines preconditions, numbered steps, and expected outcomes. The system passes when all journeys complete as described.

---

## J1 — Happy-path task execution with full hook coverage

**Preconditions:**
- Service is running.
- No blocked tools are configured in the seeded hook config for the `summarize-report` template.
- Mock LLM or live model is available.

**Steps:**
1. Open the App UI tab.
2. Select `summarize-report` from the Task template dropdown; click **Load seeded example** to fill the task description and tool set.
3. Leave `Simulate blocked tool` unchecked.
4. Click **Submit task**.
5. Observe the new card in the live list.
6. Within ~2 s the status pill transitions from `SUBMITTED` to `HOOK_CHAIN_READY`.
7. Within ~5 s the pill transitions to `EXECUTING`.
8. Within ~30 s the pill transitions to `COMPLETED`; an `outcome` badge of `SUCCESS` appears.
9. Within ~1 s a coverage score chip appears (score ≥ 3).
10. Click the card to open the detail pane; expand the hook-log table.

**Expected outcomes:**
- Hook-log table contains at least one `BEFORE_LLM_CALL` entry with verdict `PASS_THROUGH`.
- Hook-log table contains at least one `BEFORE_TOOL_CALL` entry with verdict `ALLOWED` for each tool called.
- Hook-log table contains at least one `AFTER_TOOL_CALL` entry with verdict `PASS_THROUGH` for each tool that returned data.
- Coverage score chip shows ≥ 3 with a note confirming paired before/after entries exist.
- The agent's response text is non-empty and summarizes the task result.

---

## J2 — Blocked tool call with graceful agent recovery

**Preconditions:**
- Service is running.
- The `calculate-metrics` seeded tool set includes one tool (`raw-database-query`) in the `blockedToolNames` list.

**Steps:**
1. Select `calculate-metrics` from the Task template dropdown; click **Load seeded example**.
2. Tick the `Simulate blocked tool` checkbox (this ensures the agent will attempt `raw-database-query` on its first iteration).
3. Click **Submit task**.
4. Wait for `COMPLETED` (or `PARTIAL`) status.
5. Open the detail pane and expand the hook-log table.

**Expected outcomes:**
- At least one hook-log entry has `hookPoint = BEFORE_TOOL_CALL`, `toolNameOrPhase = raw-database-query`, and `verdict = BLOCKED`.
- The `rejectReason` field on that entry is non-empty, naming the blocked tool.
- The `outcome` badge is `PARTIAL` or `SUCCESS` — not `FAILED` (agent recovered with an alternative tool or explained the gap).
- The agent's response text acknowledges that `raw-database-query` was unavailable and describes what it did instead.
- The UI never shows a raw error stack; the BLOCKED entry is the only visible indication of the rejection.

---

## J3 — Sensitive pattern in tool output is redacted before agent context

**Preconditions:**
- Service is running.
- The seeded `lookup-policy` tool set's mock response for `fetch-policy-text` contains a string matching the `password=...` sensitive pattern (`password=S3cr3t!`).

**Steps:**
1. Select `lookup-policy` from the Task template dropdown; click **Load seeded example**.
2. Click **Submit task**.
3. Wait for `COMPLETED` status.
4. Open the detail pane and inspect the hook-log table.

**Expected outcomes:**
- At least one hook-log entry has `hookPoint = AFTER_TOOL_CALL`, `toolNameOrPhase = fetch-policy-text`, and `verdict = REDACTED`.
- The `sanitizedPayload` field on that entry contains `[REDACTED-PASSWORD_PATTERN]` in place of the original credential string.
- The `originalPayloadHash` field is a non-empty SHA-256 hex string.
- The agent's response text does not contain `S3cr3t!` or any reconstruction of the redacted value.
- The entity audit log (accessible via `GET /api/observations/{id}`) stores only the hash, not the raw output.

---

## J4 — Credential token in assembled prompt is stripped before model invocation

**Preconditions:**
- Service is running.
- The `calculate-metrics` seeded task includes a context note containing a Bearer token (`Bearer eyJhbGciOiJSUzI1NiJ9...`) that was injected into the task description via the seeded example.

**Steps:**
1. Select `calculate-metrics` from the Task template dropdown; click **Load seeded example** (this loads the variant with an injected token).
2. Click **Submit task**.
3. Wait for `COMPLETED` status.
4. Open the detail pane and inspect the hook-log table.

**Expected outcomes:**
- At least one hook-log entry has `hookPoint = BEFORE_LLM_CALL` and `verdict = REDACTED`.
- The `sanitizedPayload` field on that entry indicates `[REDACTED-CREDENTIAL]` replaced the Bearer token.
- The LLM call log (if visible in the model provider's debug output) shows `[REDACTED-CREDENTIAL]` in the assembled prompt at the position where the token was.
- The agent's response text does not reproduce the Bearer token.
- If the `rejectReason` field is populated, it notes that credential stripping occurred — not that the task failed.

---

## J5 — Low coverage score flags incomplete hook chain

**Preconditions:**
- Service is running.
- A task is submitted with a tool set where the mock response returns an `AgentOutcome` whose `hookLog` is missing `AFTER_TOOL_CALL` entries (the mock omits them to simulate a misconfigured hook chain).

**Steps:**
1. Submit any task template.
2. Wait for `COMPLETED` status.
3. Observe the coverage score chip on the card.

**Expected outcomes:**
- The `CoverageScorer` detects that one or more tool calls lack a matching `AFTER_TOOL_CALL` entry.
- The coverage score chip shows ≤ 2.
- The card border is highlighted in red.
- The coverage note text states which hook pairs are missing (e.g., "Missing AFTER_TOOL_CALL entries for: read-file").
- The `COMPLETED` status is still shown (a coverage score gap does not make the observation `FAILED`); it is a signal for the operator to inspect.

---

## J6 — Multiple simultaneous tasks processed independently

**Preconditions:**
- Service is running.

**Steps:**
1. Submit three different task templates in quick succession without waiting for each to complete.
2. Observe the live list: three cards appear, each with its own status pill.
3. Wait for all three to reach `COMPLETED`.

**Expected outcomes:**
- Each observation has a distinct `observationId`.
- Hook logs in each card pertain only to that observation's tool calls — no cross-contamination.
- Coverage scores are independent; one low-scoring observation does not affect the others.
- The SSE stream delivers individual `observation-update` events per observation, identified by `observationId`.
