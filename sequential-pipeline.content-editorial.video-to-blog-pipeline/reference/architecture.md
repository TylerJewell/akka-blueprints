# Architecture — video-to-blog-pipeline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs four tasks in sequence. `BlogEndpoint` accepts a `{videoUrl}` POST, writes `PostCreated` onto `BlogPostEntity`, and starts `BlogPipelineWorkflow` keyed by `"pipeline-" + postId`. The workflow's first step (`transcriptStep`) emits `TranscriptStarted`, then calls `BlogAgent` with `TaskDef.taskType(EXTRACT_TRANSCRIPT)` and a `phase = TRANSCRIPT` metadata tag. The agent invokes `TranscriptTools.fetchTranscript` and `TranscriptTools.extractChapters`; once the agent returns a `Transcript`, the workflow writes `TranscriptExtracted` onto the entity and advances to `summaryStep`. The same pattern repeats for `summaryStep` (SUMMARISE task, `SummaryTools`), `draftStep` (DRAFT task, `DraftTools`), and `polishStep` (POLISH task, `PolishTools`). On the POLISH task, after the agent's tools have all run, `PublishGuardrail` intercepts the final response text before `PostPolished` is recorded — this is the `before-agent-response` hook. After `PostPolished` lands, `evalStep` runs `EditorialScorer` over the recorded `(Transcript, VideoSummary, BlogDraft, BlogPost)` quad — no LLM call — and writes `EvaluationScored`. `BlogPostView` projects every event into a read-model row; `BlogEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `EditorialScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `transcriptStep` and `summaryStep`, the workflow writes `TranscriptExtracted` onto the entity. The next step then reads `transcript` from the entity to build the SUMMARISE task's instruction context. The agent never sees transcript-phase context inside the summarise task's conversation; the typed handoff is the only path information travels.
2. The publish guardrail fires on the final POLISH response, not on a tool call. The agent has already invoked all four `PolishTools` methods before the guardrail runs. This placement means the guardrail sees the finished prose — the output a reader would encounter — rather than an intermediate tool result.

The agent calls are bounded by per-step timeouts (90 s on `transcriptStep`, 60 s on `summaryStep`, `draftStep`, and `polishStep`). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Eleven states. The interesting paths:

- The happy path walks `CREATED → TRANSCRIBING → TRANSCRIBED → SUMMARISING → SUMMARISED → DRAFTING → DRAFTED → POLISHING → POLISHED → EVALUATED`.
- Four failure transitions land in `FAILED`: an agent error or exhausted retry budget during `TRANSCRIBING`, `SUMMARISING`, `DRAFTING`, or `POLISHING`. A `FAILED` post's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. The agent retries the POLISH task within its 3-iteration budget; only exhausting that budget or hitting the step timeout transitions to `FAILED`.

There is no `PUBLISHED` state. The blog post is an editorial draft that a human editor reviews before scheduling. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`BlogPostEntity` is the source of truth. It emits twelve event types — four lifecycle starts, four lifecycle completions, the evaluation, the guardrail audit, the failure, and the initial creation. `BlogPostView` projects every event into a row used by the UI. `BlogPipelineWorkflow` both reads (`getPost`) and writes (`startTranscript`, `recordTranscript`, `startSummary`, `recordSummary`, `startDraft`, `recordDraft`, `startPolish`, `recordPost`, `recordEvaluation`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `BlogAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any post that lands in the entity log, the video content passed through:

1. **Publish guardrail** — every POLISH task's final response is screened for prohibited content before `PostPolished` is recorded. A response containing a prohibited phrase is rejected; a `GuardrailRejected` event records the violation for audit. The agent retries; only exhausting the retry budget transitions to `FAILED`.
2. **BlogAgent (4 task runs)** — four model calls, four structured outputs. Each task's typed result is the dependency handoff to the next phase. The agent is stateless across phases; typed events on the entity are the only memory.
3. **On-decision evaluator** — every emitted post gets a 1–5 editorial quality score. Section parity, heading completeness, word count, and body completeness are each worth one point on a base of 1.

Each step is independent. The guardrail does not check structural quality; the evaluator does not check prohibited content. Removing one of them opens an explicit gap the other does not silently cover.
