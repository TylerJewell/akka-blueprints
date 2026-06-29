# User journeys — akka-async-agent-mesh

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path: message dispatched and processed

**Preconditions:** Service running on declared port (9497); a model-provider key set, or the mock LLM selected during scaffolding. Both agent instances (`agent-alpha`, `agent-beta`) are registered in the allowed roster.

**Steps:**
1. Open `http://localhost:9497/`. The App UI tab is visible.
2. In the form, select "agent-alpha" as From, "agent-beta" as To, enter topic "status-query" and a message body. Click Send.
3. A `requestId` is returned; a message card appears in the `Dispatched` column.

**Expected:**
- Within a few seconds the dispatch workflow fires: the card moves to `Delivered` as the receipt is recorded.
- The processing workflow starts asynchronously: the card moves to `Processing`, then to `Processed` with a processing summary.
- The `agent-beta` memory panel gains one or more new `MemoryEntry` items.
- The board updates live via SSE throughout, with no page reload.

## J2 — Async fire-and-forget: dispatcher does not block processor

**Preconditions:** As J1.

**Steps:**
1. Trigger a dispatch. Note the timestamp when the dispatch workflow records its delivery receipt.
2. Observe the board.

**Expected:**
- `MessageDispatchWorkflow` terminates (message status = `Delivered`) before `MessageProcessingWorkflow` reaches `Processing`. The two timelines are independent — there is no observable "wait" on the dispatch side.
- The processing result arrives asynchronously; the message eventually reaches `Processed` without any user action after the initial send.
- `GET /api/receipts/{messageId}` returns a `DeliveryReceipt` even while the message is still `Processing`.

## J3 — PII sanitizer: personal data redacted before storage

**Preconditions:** As J1. Under the mock LLM, `dispatch-scenarios.jsonl` includes a scenario whose body contains an email address; with a real model, a message body containing a recognizable email.

**Steps:**
1. Trigger the dispatch scenario whose body contains PII (e.g., `body: "Please contact alice@example.com for the report."`).
2. Watch the message reach `Processed`.

**Expected:**
- `GET /api/messages/{id}` returns the message with `processingSummary` written; the `body` field in the view row does not contain the original email address — it contains `[REDACTED:EMAIL]`.
- `GET /api/agents/agent-beta/memory` does not contain the original email address in any `MemoryEntry`.
- No raw PII appears in any SSE event payload.

## J4 — Guardrail blocks send to unknown recipient

**Preconditions:** As J1. Under the mock LLM, `dispatch-scenarios.jsonl` includes a scenario whose `toAgent` is `"agent-gamma"` (not in the roster).

**Steps:**
1. Trigger the dispatch scenario targeting `agent-gamma`.

**Expected:**
- The before-tool-call guardrail refuses the `send_message_to_agent_async` call.
- No `MessageEntity` advances past `SEND_BLOCKED`; no processing workflow starts.
- The message card on the board shows status `Send Blocked` with the guardrail reason naming the unknown recipient.
- `GET /api/messages/{id}` returns `blockedReason` populated and all post-dispatch fields `null`.

## J5 — Operator halt and resume

**Preconditions:** As J1, with at least one scenario triggerable.

**Steps:**
1. Click Halt in the control strip (or `POST /api/control/halt` with a reason and actor).
2. Trigger a dispatch while halted.
3. Observe for ~10 s.
4. Click Resume (or `POST /api/control/resume`).

**Expected:**
- After Halt: `GET /api/control` reports `halted: true`; the red halted banner appears. Any dispatch attempt while halted results in `SEND_BLOCKED` (the G1 guardrail reads the halt flag).
- After Resume: `halted` returns to `false`; new dispatches proceed normally and previously queued scenarios continue through the normal lifecycle.
