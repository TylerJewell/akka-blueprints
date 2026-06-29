# Architecture — scrum-master-bot

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph centres on one decision-making LLM call. `StandupEndpoint` accepts a session-start request and writes a `SprintActivated` event onto `StandupEntity`. The `SprintConsumer` Consumer subscribes, builds the sprint context (roster, authorized ticket ids, sprint metadata), and writes it back via `attachSprintContext`. The same Consumer then starts a `StandupWorkflow` instance. The workflow's `standupStep` calls `ScrumMasterAgent` — the single AutonomousAgent — with the sprint context as `TaskDef.instructions(...)`. During its task, the agent calls the `postTicketUpdate` tool; each call passes through `TicketWriteGuardrail` before reaching `TicketPostingService`. Once the agent returns its `StandupSummary`, the workflow records it, then runs `postStep` to deliver any remaining ticket updates. `StandupView` projects every entity event into a read-model row; `StandupEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `TicketPostingService` is a plain service class — not an agent. The guardrail (`TicketWriteGuardrail`) is also a supporting class, not an LLM call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments where the system waits:

1. The `SprintConsumer` subscription lag between `SprintActivated` and `SprintContextAttached` — sub-second in normal operation.
2. The `collectSprintStep` polling loop inside the workflow — polls `StandupEntity` every 1 s up to its 15 s timeout, advancing as soon as `session.sprintContext().isPresent()` returns true.

The agent call itself is bounded by `standupStep`'s 120 s timeout. This budget accommodates multiple tool-call iterations: the agent may call `postTicketUpdate` several times within one task, each gated by `TicketWriteGuardrail`. `postStep` is synchronous and finishes in milliseconds — `TicketPostingService` is in-process.

## State machine

Five states. The notable paths:

- The happy path is `COLLECTING → RUNNING → SUMMARY_READY → POSTED`.
- `COLLECTING` briefly revisits itself on `SprintContextAttached` — the entity stays in `COLLECTING` until `StandupStarted` fires, because the context may arrive slightly after the sprint is activated.
- Two failure transitions land in `FAILED`: a context error during `COLLECTING`, and an agent error (or budget exhaustion) during `RUNNING`. A `FAILED` session retains all data collected up to that point for manual inspection.
- There is no automatic retry at the session level. If a session fails, the user starts a new one. The workflow's `defaultStepRecovery` handles transient errors within a step.

## Entity model

`StandupEntity` is the source of truth. It emits six event types. `StandupView` projects every event into a row used by the UI. `SprintConsumer` subscribes to entity events to load sprint context. `StandupWorkflow` both reads (`getSession`) and writes (`markRunning`, `recordSummary`, `recordPostResult`, `fail`) on the entity. The relationship between `ScrumMasterAgent` and `StandupSummary` is "returns" — the agent's task result is the summary record.

## Governance flow

For any ticket update that lands in the entity log, the write passed through:

1. **ScrumMasterAgent** — one model call, one structured output (the `StandupSummary`).
2. **before-tool-call guardrail** — out-of-scope ticket ids are rejected before `TicketPostingService` is ever called. Only authorized ids reach the simulated ticket client.

The guardrail is the sole governance mechanism in this baseline. It is targeted and narrow: it checks one property (ticket id membership) on one tool call (`postTicketUpdate`). Adding a second control — for example, a `before-agent-response` check that verifies every `memberUpdates[].memberId` matches a roster entry — would follow the same pattern and is noted in `eval-matrix.yaml` as an extension point.
