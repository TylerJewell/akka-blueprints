# Architecture — nl2sql

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call and one Postgres connection. `QueryEndpoint` accepts a question submission and writes a `QuerySubmitted` event onto `QueryEntity`. The `SchemaRegistryConsumer` Consumer subscribes, looks up the relevant schema tables from the in-process `SchemaRegistry`, and writes the schema fragment back via `attachSchema`. The same Consumer then starts a `QueryWorkflow` instance. The workflow's `translateAndRunStep` calls `SqlQueryAgent` — the single AutonomousAgent — with the user question and schema context in the task instructions. The agent calls `SchemaLookupTool` to confirm column details, then calls `QueryExecutionTool` with a generated SELECT. Before `QueryExecutionTool` fires, `SqlSafetyGuardrail` inspects the SQL; if unsafe, it rejects and the agent retries. If a write/DDL keyword is detected, `SqlSafetyHalt` terminates the task before the database is touched. Once a result passes all checks, the workflow writes `ResultRecorded` and runs `QueryResultScorer` in `scoreStep`. The score lands as `QueryScored`. `QueryView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The result scorer (`QueryResultScorer`) is a deterministic rule-based component — not an LLM. Exactly one component talks to a model, which is what makes this a faithful **single-agent** example.

## Interaction sequence

The sequence traces the happy path (J1). Note the two-stage pre-execution check:

1. The `SchemaRegistryConsumer` subscription lag between `QuerySubmitted` and `SchemaAttached` — sub-second in normal operation.
2. The `awaitSchemaStep` polling loop inside the workflow — polls `QueryEntity` every 1 s up to its 10 s timeout.

The agent's tool loop may iterate multiple times before the guardrail accepts a query (G1 path). The `scoreStep` is synchronous and finishes in milliseconds.

## State machine

Seven states. Notable paths:

- The happy path is `SUBMITTED → SCHEMA_ATTACHED → TRANSLATING → RESULT_READY → SCORED`.
- `HALTED` is a terminal state distinct from `FAILED`: it means the agent generated a write or DDL statement and the safety halt fired. The partial state is preserved on the entity so the user can inspect what was attempted.
- `FAILED` covers infrastructure problems: schema lookup error, Postgres unreachable, workflow timeout after retries exhausted.
- There is no `APPROVED` state. A read query either produces a result or it does not; no human approval gate sits between the result and the UI.

## Entity model

`QueryEntity` is the source of truth. It emits seven event types. `QueryView` projects every event into a row used by the UI. `SchemaRegistryConsumer` subscribes to entity events to attach the schema fragment. `QueryWorkflow` both reads (`getQuery`) and writes (`markTranslating`, `recordResult`, `recordScore`, `halt`, `fail`) on the entity. The relationship between `SqlQueryAgent` and `QueryResult` is "returns" — the agent's task result is the query result record.

## Defence-in-depth governance flow

For any result that lands in the entity log, the generated SQL passed through:

1. **SqlSafetyGuardrail (G1)** — write/DDL keywords and SELECT * without LIMIT are caught before `QueryExecutionTool` fires; the agent may self-correct on retry.
2. **SqlSafetyHalt (H1)** — same keyword check as G1, independent execution. On detection the task terminates immediately; no retry. Acts as the backstop for cases where G1 would have allowed a retry but the intent is clearly unsafe.
3. **Read-only Postgres role** — the JDBC connection itself is provisioned with SELECT-only grants. Even if G1 and H1 both miss a mutation statement, the database would reject it.
4. **QueryResultScorer** — every well-formed result receives a 1–5 score so the user sees a signal about result quality (empty result set, overly broad scan, missing expected columns).

Each layer is independent. The three database-safety layers form a defence-in-depth stack; the scorer is an orthogonal quality signal.
