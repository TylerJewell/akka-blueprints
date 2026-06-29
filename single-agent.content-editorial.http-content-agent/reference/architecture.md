# Architecture — http-content-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`.
Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ContentEndpoint` accepts a content-brief
submission, creates a `ContentJobEntity` record, and immediately starts a `ContentWorkflow`.
The workflow's `generateStep` calls `ContentGeneratorAgent` — the single AutonomousAgent — with
the formatted brief as `TaskDef.instructions(...)`. The agent's `before-agent-response` guardrail
(`DraftGuardrail`) validates each candidate response before it leaves the agent loop. Once a draft
passes, the workflow calls `ContentJobEntity.approveContent(content)` and the entity transitions
to `APPROVED`. `ContentView` projects every entity event into a read-model row; `ContentEndpoint`
serves the read model to the UI over REST and SSE.

There is no Consumer in this blueprint. The submission → generation path is synchronous inside
the workflow; the guardrail is the only asynchronous checkpoint. That is what makes this a
lean single-agent example: one HTTP boundary, one agent, one guard.

## Interaction sequence

The sequence traces the happy path (J1). One moment of latency:

1. The `generateStep` call to `ContentGeneratorAgent` — bounded by the 90 s step timeout,
   which accommodates LLM generation time for longer-form content (BLOG_POST at 600 words can
   take 10–20 s on real providers).

The guardrail check is synchronous within the agent's turn cycle. If the draft fails, the
structured-error response goes back to the agent immediately, and the retry starts without any
external I/O between iterations.

## State machine

Four states. The paths:

- The happy path is `REQUESTED → GENERATING → APPROVED`.
- One failure transition lands in `FAILED`: an agent error or guardrail-exhaustion (all 3
  iterations fail validation) during `GENERATING`. A `FAILED` job's prior data is preserved on
  the entity — the UI shows the partial state for the submitter.
- There is no `PUBLISHED` state. The generated content is advisory output; the caller decides
  what to do with it. The blueprint deliberately stops at `APPROVED`.

## Entity model

`ContentJobEntity` is the source of truth. It emits four event types. `ContentView` projects
every event into a row used by the UI. `ContentWorkflow` both reads and writes on the entity
(`startGeneration`, `approveContent`, `failJob`). The relationship between
`ContentGeneratorAgent` and `GeneratedContent` is "returns" — the agent's task result is the
content record stored on the entity.

## Guardrail governance flow

For any content piece that lands in the entity log, the draft passed through:

1. **ContentGeneratorAgent** — one model call, one structured output.
2. **before-agent-response guardrail** (`DraftGuardrail`) — structural completeness, word-count
   tolerance, and prohibited-word scan are all checked before the response leaves the agent loop.

Each check is independent. If the word-count check passes but the prohibited-word check fails,
the rejection message names only the prohibited-word check, giving the agent precise guidance for
its retry rather than forcing a full regeneration.
