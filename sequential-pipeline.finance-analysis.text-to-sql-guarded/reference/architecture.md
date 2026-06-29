# Architecture — text-to-sql-guarded

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `QueryEndpoint` accepts a `{question}` POST, writes `QueryCreated` onto `QueryEntity`, and starts `QueryPipelineWorkflow` keyed by `"pipeline-" + queryId`. The workflow's first step (`parseStep`) emits `ParseStarted`, then calls `SqlAgent` with `TaskDef.taskType(PARSE_QUESTION)` and a `phase = PARSE` metadata tag. The agent invokes `ParseTools.resolveSchema` and `ParseTools.generateSql`; every call passes through `SqlGuardrail` first.

Once the agent returns a `ParsedQuery`, the workflow immediately runs `SqlSafetyInspector.inspect(sql)` — a deterministic in-process check for destructive statement patterns. If the inspection returns `safe = false`, the workflow writes `SafetyHaltFired{sql, matchedPattern}`, transitions the entity to `HALTED`, and ends via `haltStep`. The database is never contacted.

If the SQL is safe, the workflow writes `QuestionParsed` and advances to `queryStep`. Same pattern: the QUERY task carries `phase = QUERY` metadata. After the agent returns a `RawResult`, the workflow runs `PiiSanitizer.sanitize(rawResult)` to produce a `SanitizedResult` before calling `formatStep`. The FORMAT task's instruction context carries the sanitized rows only; the LLM never sees raw PII values.

After `ResultsFormatted` lands, `QueryView` projects every event into a read-model row. `QueryEndpoint` serves the read model over REST and SSE.

The graph has exactly one LLM-calling component. `SqlSafetyInspector` and `PiiSanitizer` are deterministic — that is what makes this a faithful **single-agent sequential-pipeline** example with defence-in-depth governance.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. **The task boundary IS the dependency contract.** Between `parseStep` and `queryStep`, the workflow writes `QuestionParsed` onto the entity. The next step reads `parsedQuery` from the entity to build the QUERY task's instruction context. The agent never sees parse-phase context inside the query task's conversation; the typed handoff is the only path information travels.
2. **Every tool call is filtered through `SqlGuardrail`.** The guardrail reads the in-flight task's `phase` metadata and the current `QueryEntity.status`, and applies the per-phase accept matrix. A misordered call is rejected before the tool body executes. This is the same mechanism as in any sequential-pipeline blueprint; what differs here is the financial risk of a misordered `executeSql` call that reaches the database while the safety inspection has not yet run.
3. **`PiiSanitizer` runs at the workflow level, not inside the agent.** The sanitizer sees the full `RawResult` before the FORMAT agent task starts. This guarantees that neither the LLM's prompt context nor the stored `QueryReport` contain raw PII — regardless of what the model might do with that data.

The agent calls are bounded by per-step timeouts (60 s on parse / query / format). `haltStep` is synchronous and finishes in milliseconds.

## State machine

Ten states. The interesting paths:

- The happy path walks `CREATED → PARSING → PARSED → QUERYING → QUERIED → FORMATTING → FORMATTED`.
- `HALTED` is reached from `PARSING` when `SqlSafetyInspector` fires. It is a clean terminal state — data up to `ParsedQuery` is preserved on the entity; the halt reason and matched pattern are surfaced in the UI.
- Three failure transitions land in `FAILED`: an agent error during `PARSING`, `QUERYING`, or `FORMATTING`. A failed query's partial data is preserved; the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `PUBLISHED` state. The report is advisory; the analyst consumes it and acts outside the system.

## Entity model

`QueryEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the safety halt, the guardrail audit, the failure, and the initial creation. `QueryView` projects every event into a row used by the UI. `QueryPipelineWorkflow` both reads and writes on the entity across five commands: `recordParsed`, `recordHalt`, `recordQueryResult`, `recordReport`, `recordGuardrailRejection`.

The relationship between `SqlAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload. `PiiSanitizer` sits between `queryStep`'s result and `formatStep`'s task context; its output (`SanitizedResult`) is embedded in `QueryReport` but the sanitizer itself has no Akka primitive — it is a plain Java class invoked synchronously by the workflow.

## Defence-in-depth governance flow

For any query that lands in the entity log, the question passed through:

1. **Phase-gate guardrail** — every tool call is filtered. A QUERY-phase tool called during PARSE is rejected before the tool body runs; a `GuardrailRejected` event records the violation. Critically, `executeSql` cannot be called before the SQL has been safety-inspected.
2. **Automatic safety halt** — the generated SQL is inspected synchronously before `queryStep` starts. A destructive pattern ends the workflow immediately; the database is never contacted.
3. **PII sanitizer** — raw result rows are redacted before the FORMAT agent task and before storage. The report and all downstream surfaces carry only sanitized values.
4. **SqlAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.

Each mechanism is independent. Removing the safety halt does not affect the guardrail; removing the sanitizer does not affect the halt. The governance coverage is additive.
