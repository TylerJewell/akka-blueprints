# Architecture — siem-enrichment

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `AlertEndpoint` accepts a `{rawAlertJson}` POST, writes `AlertReceived` onto `AlertEntity`, and starts `AlertEnrichmentWorkflow` keyed by `"pipeline-" + alertId`. The workflow's first step (`fetchStep`) emits `FetchStarted`, then calls `AlertEnrichmentAgent` with `TaskDef.taskType(FETCH_ALERT)` and a `phase = FETCH` metadata tag. The agent invokes `FetchTools.fetchAlertDetail` and `FetchTools.fetchHostContext`; every call passes through `TicketScopeGuardrail` first. Once the agent returns an `AlertDetail`, the workflow writes `FetchCompleted` onto the entity and advances to `enrichStep` — same pattern, the ENRICH task carries `phase = ENRICH`. Then `ticketStep` runs with `phase = TRIAGE`. After `TicketCreated` lands, `evalStep` runs `TriageQualityScorer` over the recorded `(AlertDetail, EnrichedAlert, TriageTicket)` triple — no LLM call — and writes `EvaluationScored`. `AlertView` projects every event into a read-model row; `AlertEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `TriageQualityScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `fetchStep` and `enrichStep`, the workflow writes `FetchCompleted` onto the entity. The next step then reads `alertDetail` from the entity to build the ENRICH task's instruction context. The agent never sees fetch-phase context inside the enrich task's conversation; the typed handoff is the only path information travels.
2. Every tool call is filtered through `TicketScopeGuardrail`. The guardrail reads the in-flight task's `phase` metadata and the current `AlertEntity.status`, and applies the per-phase accept matrix. A TRIAGE-phase tool called before `EnrichmentProduced` has been recorded is rejected before the tool body executes — and, importantly, before any network call reaches Zendesk.

The agent calls themselves are bounded by per-step timeouts (60 s on fetch / enrich / triage). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `RECEIVED → FETCHING → FETCHED → ENRICHING → ENRICHED → TRIAGING → TRIAGED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `FETCHING`, `ENRICHING`, or `TRIAGING`. A `FAILED` alert's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `CLOSED` or `RESOLVED` state. The triage ticket is advisory; the SOC analyst acts on it outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`AlertEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the evaluation, the guardrail audit, the failure, and the initial reception. `AlertView` projects every event into a row used by the UI. `AlertEnrichmentWorkflow` both reads (`getAlert`) and writes (`startFetch`, `recordAlertDetail`, `startEnrich`, `recordEnrichment`, `startTriage`, `recordTriageTicket`, `recordEvaluation`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `AlertEnrichmentAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any alert that lands in the entity log, the raw JSON passed through:

1. **Ticket-scope guardrail** — every tool call is filtered. A TRIAGE-phase tool called during ENRICH is rejected before the tool body runs and before any Zendesk network call is made; a `GuardrailRejected` event records the violation for audit.
2. **AlertEnrichmentAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-incident evaluator** — every emitted triage ticket gets a 1–5 quality score. ATT&CK technique presence, technique validity, severity accuracy, and routing completeness are each worth one point on a base of 1.

Each layer is independent. The guardrail does not check technique validity; the evaluator does not check phase order. Removing one opens an explicit gap the other does not silently cover.
