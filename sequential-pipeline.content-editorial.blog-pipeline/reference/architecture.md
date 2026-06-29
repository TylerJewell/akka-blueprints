# Architecture — AI Blog Writer Pipeline with Ollama

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs five tasks in sequence. `PostEndpoint` accepts a `{topic, postType}` POST, writes `PostCreated` onto `PostEntity`, and starts `BlogPipelineWorkflow` keyed by `"pipeline-" + postId`. The workflow's first step (`researchStep`) emits `ResearchStarted`, then calls `BlogWriterAgent` with `TaskDef.taskType(RESEARCH_TOPIC)` and a `phase = RESEARCH` metadata tag. The agent invokes `ResearchTools.searchReferences` and `ResearchTools.fetchSummary`; when the agent returns a `ResearchNotes`, the workflow passes the prose to `ContentPolicyGuardrail` before writing `ResearchCollected` onto the entity. The same pattern repeats through `outlineStep`, `draftStep`, `editStep`, and `publishStep`. After `PostPublished` lands, the post is visible in the UI as a fully assembled typed `Post`. `PostView` projects every event into a read-model row; `PostEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `ContentPolicyGuardrail` is a deterministic rule-based checker; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the editorial dependency contract. Between `researchStep` and `outlineStep`, the workflow writes `ResearchCollected` onto the entity. The next step reads `research` from the entity to build the OUTLINE task's instruction context. The agent never sees research-phase context inside the outline task's conversation; the typed handoff is the only path information travels.
2. Every agent response is filtered through `ContentPolicyGuardrail`. The guardrail checks prohibited patterns, originality, and post-type alignment. A non-compliant response is rejected before the result event is written onto the entity.

The agent calls are bounded by per-step timeouts (90 s on each of the five phases, sized for Ollama's local inference latency). `error` step is synchronous and finishes in milliseconds.

## State machine

Twelve states. The interesting paths:

- The happy path walks `CREATED → RESEARCHING → RESEARCHED → OUTLINING → OUTLINED → DRAFTING → DRAFTED → EDITING → EDITED → PUBLISHING → PUBLISHED`.
- Five failure transitions land in `FAILED`: a step exhausting its 4-iteration guardrail retry budget, or a step timeout on any of the five phases. A `FAILED` post's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailBlocked` and `ContentCleared` are side-events recorded for audit; they do not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `DISTRIBUTED` state. The post is advisory content; an editor reviews and distributes it outside the system. The blueprint deliberately stops at `PUBLISHED`.

## Entity model

`PostEntity` is the source of truth. It emits fourteen event types — five lifecycle starts, five lifecycle completions, the content-cleared audit event, the guardrail-blocked audit event, the publish confirmation, and the failure. `PostView` projects every event into a row used by the UI. `BlogPipelineWorkflow` both reads (`getPost`) and writes (`startResearch`, `recordResearch`, `startOutline`, `recordOutline`, `startDraft`, `recordDraft`, `startEdit`, `recordEditedDraft`, `startPublish`, `recordContentCleared`, `recordPost`, `recordGuardrailBlock`, `fail`) on the entity. The relationship between `BlogWriterAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload after the guardrail clears it.

## Content-policy governance flow

For any post that lands in the entity log, each phase's prose passed through:

1. **Before-agent-response guardrail** — every typed task response is checked before the result event is written. Non-compliant prose is rejected with a structured reason; `GuardrailBlocked` records the violation for audit; the agent retries.
2. **BlogWriterAgent (5 task runs)** — five model calls, five structured outputs. Each task's typed result is the dependency handoff to the next phase.

The guardrail covers all five phases with a single registered hook. A phase-level check failure opens an explicit gap in the post's audit log that the deployer can observe in the `GuardrailBlocked` event stream.
