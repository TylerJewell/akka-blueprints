# Architecture — blog-writer-pipeline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `BlogPostEndpoint` accepts a `{topic, style}` POST, writes `PostCreated` onto `BlogPostEntity`, and starts `BlogWritingWorkflow` keyed by `"workflow-" + postId`. The workflow's first step (`researchStep`) emits `ResearchStarted`, then calls `BlogWriterAgent` with `TaskDef.taskType(RESEARCH_TOPIC)` and a `phase = RESEARCH` metadata tag. The agent invokes `ResearchTools.searchTopicReferences` and `ResearchTools.fetchKeyPoints`. Once the agent returns `ResearchNotes`, the workflow writes `ResearchCollected` onto the entity and advances to `outlineStep` — same pattern, the OUTLINE task carries `phase = OUTLINE`. Then `draftStep` runs with `phase = DRAFT`. After the agent's DRAFT response passes `BrandGuardrail`, the workflow writes `DraftWritten`. Then `qualityCheckStep` runs `QualityScorer` over the recorded `(ResearchNotes, Outline, BlogPost)` triple — no LLM call — and writes `QualityChecked`. `BlogPostView` projects every event into a read-model row; `BlogPostEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `QualityScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `researchStep` and `outlineStep`, the workflow writes `ResearchCollected` onto the entity. The next step then reads `research` from the entity to build the OUTLINE task's instruction context. The agent never sees research-phase context inside the outline task's conversation; the typed handoff is the only path information travels.
2. The brand guardrail fires on every DRAFT response before `DraftWritten` is emitted. The guardrail reads the candidate `BlogPost` as a complete typed record and applies three deterministic checks. A non-conformant draft is rejected before the event is written; only brand-conformant posts enter the event log.

The agent calls are bounded by per-step timeouts (60 s on research / outline, 90 s on draft to accommodate guardrail retry cycles). `qualityCheckStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → RESEARCHING → RESEARCHED → OUTLINING → OUTLINED → DRAFTING → DRAFTED → QUALITY_CHECKED`.
- Three failure transitions land in `FAILED`: an agent error during `RESEARCHING`, `OUTLINING`, or `DRAFTING`. A `FAILED` post's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `PUBLISHED` state. The post is advisory editorial output; a human editor reviews and publishes it outside the system. The blueprint deliberately stops at `QUALITY_CHECKED`.

## Entity model

`BlogPostEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the quality evaluation, the guardrail audit, the failure, and the initial creation. `BlogPostView` projects every event into a row used by the UI. `BlogWritingWorkflow` both reads (`getPost`) and writes (`startResearch`, `recordResearch`, `startOutline`, `recordOutline`, `startDraft`, `recordDraft`, `recordQuality`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `BlogWriterAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any post that lands in the entity log, the topic passed through:

1. **Brand-voice guardrail** — every DRAFT agent response is filtered before it becomes a `DraftWritten` event. A draft that is too short, contains a forbidden phrase, or lacks a CTA is rejected before it enters the event log; a `GuardrailRejected` event records the violation for audit.
2. **BlogWriterAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-decision evaluator** — every accepted draft gets a 1–5 quality score. Section coverage, section depth, title fidelity, and section parity are each worth one point on a base of 1.

Each step is independent. The guardrail does not check quality dimensions; the evaluator does not check brand-voice rules. Removing one of them opens an explicit gap the other does not silently cover.
