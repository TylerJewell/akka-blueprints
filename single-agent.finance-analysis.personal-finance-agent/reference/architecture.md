# Architecture — personal-finance-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a query submission and writes a `QuerySubmitted` event onto `QueryEntity`. The `TransactionSanitizer` Consumer subscribes, tokenises PII in the raw transaction data, and writes the sanitized context back via `attachSanitized`. The same Consumer then starts a `QueryWorkflow` instance. The workflow's `answerStep` calls `FinanceAssistantAgent` — the single AutonomousAgent — with the user's query text and sanitized account context. The agent calls `FinanceTools` as needed; write tools (`transferFunds`, `payBill`) pass through `WriteGuardrail` before execution; read tools bypass it. Once the agent returns an `AssistantResponse`, the workflow writes `AnswerRecorded` and calls `finish`. `QueryView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `WriteGuardrail` is a synchronous validation hook, not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the spending-summary happy path (J1). Two distinct waits exist in the flow:

1. The `TransactionSanitizer` subscription lag between `QuerySubmitted` and `TransactionsSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop — polls `QueryEntity` every 1 s up to its 15 s timeout, advancing as soon as `query.sanitized().isPresent()` returns true.

The agent call is bounded by `answerStep`'s 90 s timeout. The agent may make 2–4 tool-round-trips within that window; each tool call is synchronous from the workflow's perspective. `WriteGuardrail` adds negligible latency on read calls (pass-through with no check); on write calls it performs three in-memory rule evaluations before returning.

## State machine

Five states. The interesting paths:

- Happy path: `SUBMITTED → SANITIZED → ANSWERING → ANSWERED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED` (malformed transaction data), and an agent error (timeout, tool exhaustion, or unrecoverable model error) during `ANSWERING`. A `FAILED` query's prior partial data — the sanitized context and any tool calls that completed before failure — is preserved on the entity for the user to inspect.
- There is no `APPROVED` or `PENDING` state. Write operations are either executed or blocked within the same agent task; no asynchronous approval workflow exists.

## Entity model

`QueryEntity` is the source of truth. It emits five event types. `QueryView` projects every event into a row used by the UI. `TransactionSanitizer` subscribes to entity events to produce the sanitized context. `QueryWorkflow` both reads (`getQuery`) and writes (`markAnswering`, `recordAnswer`, `finish`, `fail`) on the entity. `FinanceAssistantAgent` produces `AssistantResponse` as its task result; `FinanceTools` produces `ToolCallRecord` entries that are embedded in the response. `WriteGuardrail` decides `WriteOutcome` for each write-tool invocation.

## Governance flow for write operations

For any write operation that reaches the simulated bank state, the request passed through:

1. **PII sanitizer** — the model received tokenised transaction data; it never saw raw account numbers or full names.
2. **FinanceAssistantAgent** — one model call composed the tool invocation.
3. **before-tool-call guardrail** — balance, ownership, and principal checks validated the intent before the tool body ran.

A write that fails step 3 never mutates state. The model receives the refusal as a tool result and incorporates it into the user-facing answer. Each step is independent: removing the sanitizer leaves PII in the prompt; removing the guardrail allows over-balance or cross-user transfers. Neither gap is silently covered by the other.
