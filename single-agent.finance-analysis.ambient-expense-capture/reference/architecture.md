# Architecture — ambient-expense-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ExpenseEndpoint` accepts a receipt submission and writes a `ReceiptSubmitted` event onto `ExpenseEntity`. The `ReceiptSanitizer` Consumer subscribes, redacts PII, and writes the sanitized receipt back via `attachSanitized`. The same Consumer then starts an `ExpenseWorkflow` instance. The workflow's `captureStep` calls `ExpenseCaptureAgent` — the single AutonomousAgent — with the approved expense-category taxonomy as `TaskDef.instructions(...)` and the sanitized receipt as a `TaskDef.attachment(...)`. For each line item the agent submits via tool call, the `SubmissionGuardrail` intercepts the call before it executes. Passed tool calls result in line items with `APPROVED` status; rejected tool calls return a structured error to the agent loop. Once the agent returns a complete `ExpenseReport`, the workflow writes `ReportReady` and calls `markSystemSubmitted`. `ExpenseView` projects every entity event into a read-model row; `ExpenseEndpoint` serves the read model over REST and SSE.

The graph has no second agent. The submission guardrail (`SubmissionGuardrail`) is a policy-validation hook — it checks amounts, categories, currencies, and merchant names using the loaded taxonomy. No LLM call is involved. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct pauses:

1. The `ReceiptSanitizer` subscription lag between `ReceiptSubmitted` and `ReceiptSanitized` — sub-second in normal operation, as the sanitizer is in-process.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `ExpenseEntity` every 1 s up to its 15 s timeout, advancing as soon as `submission.sanitized().isPresent()` returns true.

The agent call is bounded by `captureStep`'s 60 s timeout. The `submitStep` is a short entity write — it finishes in milliseconds.

## State machine

Six states. The paths:

- The happy path is `SUBMITTED → SANITIZED → CAPTURING → REPORT_READY → SUBMITTED_TO_SYSTEM`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error (or iteration-budget exhaustion) during `CAPTURING`. A failed submission's partial data is preserved on the entity — the UI shows what was captured before the failure.
- `SUBMITTED_TO_SYSTEM` is the terminal happy state. The actual ERP sync is simulated in-process; a real deployer would extend `submitStep` to call an outbound integration.

## Entity model

`ExpenseEntity` is the source of truth. It emits six event types. `ExpenseView` projects every event into a row used by the UI. `ReceiptSanitizer` subscribes to entity events to compute the sanitized form. `ExpenseWorkflow` both reads (`getSubmission`) and writes (`markCapturing`, `recordReport`, `markSystemSubmitted`, `fail`) on the entity. The relationship between `ExpenseCaptureAgent` and `ExpenseReport` is "returns" — the agent's task result is the report record.

## Defence-in-depth governance flow

For any expense report that reaches the entity log, the receipt passed through:

1. **PII sanitizer** — the model never sees card numbers, names, or addresses; the audit log retains the raw form.
2. **ExpenseCaptureAgent** — one model call, one structured output per line item.
3. **before-tool-call guardrail** — each tool call to submit a line item is validated for category, amount, currency, and merchant before it executes. A bad call is rejected with a named failure code; the agent must address it.

Each layer is independent. Removing either one opens a gap the other does not cover: the sanitizer cannot catch over-limit amounts, and the guardrail cannot redact PII that has already been sent to the model.
