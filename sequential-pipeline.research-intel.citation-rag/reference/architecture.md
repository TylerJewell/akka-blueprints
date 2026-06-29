# Architecture — citation-rag

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `QueryEndpoint` accepts a `{queryText}` POST, writes `QueryCreated` onto `QueryEntity`, and starts `CitationQueryWorkflow` keyed by `"citation-" + queryId`. The workflow's first step (`retrieveStep`) emits `RetrieveStarted`, then calls `CitationAgent` with `TaskDef.taskType(RETRIEVE_PASSAGES)` and a `phase = RETRIEVE` metadata tag. The agent invokes `RetrieveTools.searchPassages` and `RetrieveTools.fetchPassage`. Once the agent returns a `PassageSet`, the workflow writes `PassagesRetrieved` onto the entity and advances to `attributeStep` — same pattern, the ATTRIBUTE task carries `phase = ATTRIBUTE`. Then `composeStep` runs with `phase = COMPOSE`. After the agent returns an `Answer`, `CitationGuardrail` inspects it — every claim's `citedPassageId` is validated against the recorded `PassageSet` before the answer is committed. If the guardrail accepts, `AnswerComposed` lands and `evalStep` runs `CoverageScorer` over the recorded `(PassageSet, ClaimSet, Answer)` triple — no LLM call — and writes `CitationScored`. `QueryView` projects every event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `CoverageScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `retrieveStep` and `attributeStep`, the workflow writes `PassagesRetrieved` onto the entity. The next step then reads `passages` from the entity to build the ATTRIBUTE task's instruction context. The agent never sees retrieve-phase context inside the attribute task's conversation; the typed handoff is the only path information travels.
2. The after-llm-response guardrail fires on the COMPOSE task's return, before the answer is committed. The guardrail validates citation linkages at the output boundary, not at the input boundary. This is the correct architectural cut for a citation-integrity check: the violation is in the LLM's output, not its input.

The agent calls are bounded by per-step timeouts (60 s on retrieve / attribute / compose). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → RETRIEVING → RETRIEVED → ATTRIBUTING → ATTRIBUTED → COMPOSING → COMPOSED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `RETRIEVING`, `ATTRIBUTING`, or `COMPOSING`. A `FAILED` query's prior data is preserved on the entity — the UI shows the partial state.
- `CitationGuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to FAILED.

There is no `APPROVED` or `PUBLISHED` state. The answer is advisory; the researcher consumes it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`QueryEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the citation score, the guardrail audit, the failure, and the initial creation. `QueryView` projects every event into a row used by the UI. `CitationQueryWorkflow` both reads (`getQuery`) and writes (`startRetrieve`, `recordPassages`, `startAttribute`, `recordClaims`, `startCompose`, `recordAnswer`, `recordCitationScore`, `recordCitationGuardrailRejection`, `fail`) on the entity. The relationship between `CitationAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any answer that lands in the entity log, the query passed through:

1. **Citation-gate guardrail** — fires after the COMPOSE task response, before the answer is committed. An answer with an uncited claim or a dangling `citedPassageId` is rejected before `AnswerComposed` is written; a `CitationGuardrailRejected` event records the violation for audit.
2. **CitationAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-decision evaluator** — every committed answer gets a 1–5 coverage score. Full attribution, resolved citations, non-empty answer, and no decoration citations are each worth one point on a base of 1.

Each step is independent. The guardrail does not check coverage quality; the evaluator does not block uncited claims. Removing one opens an explicit gap the other does not silently cover.
