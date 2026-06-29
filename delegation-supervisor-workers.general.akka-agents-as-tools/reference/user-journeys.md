# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a summarize task and watch it complete

**Preconditions:** service running on `http://localhost:9545/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a paragraph of text, select "summarize" from the operation dropdown, and Submit.
2. Observe the new request row via SSE.

**Expected:** the request progresses `QUEUED → IN_PROGRESS → COMPLETED` within ~60 s. The expanded row shows a `RoutingDecision` with `selectedTool="summarize"`, a `ToolResult` with a 30–80 word summary, and an assembled `TaskResult.output` matching that summary. Guardrail verdict is `"ok"`.

## J2 — Before-tool-call guardrail blocks a prohibited call

**Preconditions:** guardrail configured to block calls to a tool named `"restricted-tool"` (test fixture injected via test override or sample-tasks.jsonl entry with `operation="restricted-tool"`).

**Steps:**
1. Submit a request that causes the supervisor to route to the prohibited tool.
2. Watch the request.

**Expected:** `guardrailStep` intercepts the `RoutingDecision` before any sub-agent is dispatched. The workflow calls `TaskRequestEntity.block`; the request enters `BLOCKED` with a `failureReason`. No sub-agent runs. The blocked entry is visible in the App UI with the BLOCKED status pill.

## J3 — Sub-agent timeout moves the request to TIMED_OUT

**Preconditions:** `SummarizerAgent` step timeout set to 1 s (test override).

**Steps:**
1. Submit a summarize task.
2. Watch the request.

**Expected:** the `dispatchStep` times out, the workflow routes to `timeoutStep`, and the request enters `TIMED_OUT` with a `failureReason` that names the sub-agent. No partial output is returned. No infinite retry.

## J4 — Classify and translate operations route to the correct sub-agent

**Preconditions:** service running with a model provider.

**Steps:**
1. Submit a request with `operation="classify"`.
2. Submit a second request with `operation="translate"`.

**Expected:** the first request's `RoutingDecision.selectedTool` is `"classify"` and the `ToolResult.output` contains category labels. The second request's `selectedTool` is `"translate"` and the output contains translated text. Both reach `COMPLETED` without the guardrail triggering.

## J5 — Eval score appears beside a completed request

**Preconditions:** at least one `COMPLETED` request without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually.
2. Refresh the request row.

**Expected:** the request gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Delivery of the task result was not blocked by the eval (non-blocking).

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `RequestSimulator` drips a task from `sample-tasks.jsonl` every 60 s; each becomes a request that flows through the full pipeline. The App UI is non-empty on first load, covering all three operation types across the 8 canned entries.
