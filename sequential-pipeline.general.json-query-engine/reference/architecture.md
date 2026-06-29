# Architecture — json-query-engine

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `QueryEndpoint` accepts a `{question, docId}` POST, writes `QueryCreated` onto `QueryEntity`, and starts `JsonQueryWorkflow` keyed by `"workflow-" + queryId`. The workflow's first step (`parseStep`) emits `ParseStarted`, then calls `QueryAgent` with `TaskDef.taskType(PARSE_QUESTION)` and a `phase = PARSE` metadata tag. The agent invokes `ParseTools.identifyQueryIntent` and `ParseTools.selectRootKeys`; every call passes through `PathGuardrail` first. Once the agent returns a `ParsedQuestion`, the workflow writes `QuestionParsed` onto the entity and advances to `traverseStep` — the TRAVERSE task carries `phase = TRAVERSE`. Then `respondStep` runs with `phase = RESPOND`. After `ResponseComposed` lands, `evalStep` runs `AccuracyScorer` over the recorded `(ParsedQuestion, TraversalResult, QueryResult)` triple — no LLM call — and writes `AccuracyScored`. `QueryView` projects every event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `AccuracyScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `parseStep` and `traverseStep`, the workflow writes `QuestionParsed` onto the entity. The next step then reads `parsed` from the entity to build the TRAVERSE task's instruction context. The agent never sees parse-phase context inside the traverse task's conversation; the typed handoff is the only path information travels.
2. Every tool call is filtered through `PathGuardrail`. For path-expression tools, the guardrail validates the expression against three structural rules before the document is touched. A malformed path, unknown root key, or excessive depth is rejected before the traversal executes.

The agent calls are bounded by per-step timeouts (60 s on parse / traverse / respond). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → PARSING → PARSED → TRAVERSING → TRAVERSED → RESPONDING → RESPONDED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `PARSING`, `TRAVERSING`, or `RESPONDING`. A `FAILED` query's prior data is preserved on the entity — the UI shows the partial state, including any partial traversal that completed before the error.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `PUBLISHED` state. The query result is advisory; the reader consumes it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`QueryEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the accuracy evaluation, the guardrail audit, the failure, and the initial creation. `QueryView` projects every event into a row used by the UI. `JsonQueryWorkflow` both reads (`getQuery`) and writes (`startParse`, `recordParsed`, `startTraverse`, `recordTraversal`, `startRespond`, `recordResult`, `recordAccuracy`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `QueryAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Path-validation governance flow

For any query that lands in the entity log, every path-expression tool call passed through:

1. **Path-expression validation (syntax)** — the expression must begin with `$` and use only valid JSON-path operators. A malformed expression is rejected before the document file is opened.
2. **Root-key validation** — the first key segment after `$` must appear in `DocumentSchema.rootKeys` for the requested document. This prevents the agent from traversing keys that were not part of the document's declared schema, which would produce an empty result with no indication of why.
3. **Depth validation** — the path nesting depth must not exceed `DocumentSchema.maxTraversalDepth`. This bounds the traversal cost on deeply nested documents and prevents unbounded recursive descent via `..` operators.

Each check is independent. A path that passes syntax and root-key checks but exceeds the depth limit is still rejected. Removing one of the checks opens an explicit gap the others do not silently cover. All three checks run in the guardrail before the tool body executes, so the document file is never read for an invalid path.
