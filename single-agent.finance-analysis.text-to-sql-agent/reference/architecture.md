# Architecture — text-to-sql-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a question, mints a `queryId`, writes a `QuestionSubmitted` event onto `QueryEntity`, and immediately starts a `QueryWorkflow` instance. The workflow's `generateStep` calls `SqlGeneratorAgent` — the single AutonomousAgent — with the question as `TaskDef.instructions(...)` and the receipts schema DDL as a `TaskDef.attachment(...)`. The agent uses an `execute_sql` @Tool to query the embedded SQLite database. Before each `execute_sql` invocation, the `SqlGuardrail` before-tool-call hook inspects the SQL for destructive statements and injection patterns. If the guardrail accepts, the JDBC call fires. Once the agent returns its `QueryResult`, the workflow writes `QueryExecuted` to the entity. The `ResultSanitizer` Consumer subscribes to that event, redacts customer identifiers from the result rows, and writes `ResultSanitized` back to the entity. The `awaitSanitizedStep` detects the sanitized result and marks the workflow complete. `QueryView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The `ResultSanitizer` is a Consumer with a regex redaction pipeline — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct asynchronous gaps:

1. The `ResultSanitizer` subscription lag between `QueryExecuted` and `ResultSanitized` — sub-second in normal operation because the sanitizer logic is a local regex scan.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `QueryEntity` every 1 s up to its 15 s timeout, advancing as soon as `query.sanitized().isPresent()` returns true.

The agent call itself (including the tool call round-trip to SQLite) is bounded by `generateStep`'s 90 s timeout. Most queries complete in under 5 s when a model is available; the generous ceiling accommodates cold-start LLM latency.

## State machine

Six states. Notable paths:

- The happy path is `SUBMITTED → GENERATING → EXECUTING → EXECUTED → SANITIZED`.
- Three failure transitions land in `FAILED`: an agent error during SUBMITTED (task launch failure), guardrail-exhaustion during GENERATING (4 consecutive rejections), and a database error during EXECUTING. A `FAILED` query's partial state is preserved on the entity — the UI shows whatever was captured before the failure.
- There is no `APPROVED` state. The answer is informational; the analyst reads it and acts outside the system.

## Entity model

`QueryEntity` is the source of truth. It emits six event types. `QueryView` projects every event into a row used by the UI. `ResultSanitizer` subscribes to entity events to compute the sanitized result set. `QueryWorkflow` both reads (`getQuery`) and writes (`recordSql`, `markExecuting`, `recordResult`, `attachSanitized`, `fail`) on the entity. The relationship between `SqlGeneratorAgent` and `QueryResult` is "returns" — the agent's task result is the query result record.

## Defence-in-depth governance flow

For any answer that lands in the entity log, the data passed through:

1. **SqlGuardrail (before-tool-call)** — the model's generated SQL is inspected before the JDBC call. Destructive or injected SQL is blocked at the source; the agent retries within its iteration budget.
2. **SqlGeneratorAgent** — one model call, one tool call, one structured output.
3. **ResultSanitizer** — customer names and email addresses are replaced with redaction tokens before the result reaches the view or the SSE stream. The raw rows stay on the entity for authorised audit access.

Each step is independent. Removing the guardrail leaves the database exposed to LLM-generated destructive SQL. Removing the sanitizer exposes customer identifiers in the chat interface. Neither gap is silently covered by the other mechanism.
