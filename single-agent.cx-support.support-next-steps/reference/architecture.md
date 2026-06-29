# Architecture — support-next-steps

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `TicketEndpoint` accepts a submission and writes a `TicketSubmitted` event onto `TicketEntity`. The `TicketSanitizer` Consumer subscribes, redacts PII from the ticket body, and writes the sanitized form back via `attachSanitized`. The same Consumer then starts a `TicketWorkflow` instance. The workflow's `adviseStep` calls `ResolutionAdvisorAgent` — the single AutonomousAgent — with the ticket context as `TaskDef.instructions(...)` and two attachments: the sanitized ticket body and the product-area resolution library, both via `TaskDef.attachment(...)`. Once the recommendation returns, the workflow writes `RecommendationRecorded` and runs `RecommendationScorer` in `evalStep`. The score lands as `EvaluationScored`. `TicketView` projects every entity event into a read-model row; `TicketEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `RecommendationScorer` is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two moments where the system waits on something:

1. The `TicketSanitizer` subscription lag between `TicketSubmitted` and `TicketSanitized` — sub-second in normal operation because the Consumer runs in-process.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `TicketEntity` every 1 s up to its 15 s timeout, advancing as soon as `ticket.sanitized().isPresent()` returns true.

The agent call itself is bounded by `adviseStep`'s 60 s timeout. The `evalStep` is synchronous and runs in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → SANITIZED → ADVISING → RECOMMENDATION_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error (or iteration-budget exhaustion) during `ADVISING`. A `FAILED` ticket's prior data is preserved on the entity — the UI shows the partial state for the support agent.
- There is no `RESOLVED` or `CLOSED` state. The recommendation is advisory; the support agent reads it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`TicketEntity` is the source of truth. It emits six event types. `TicketView` projects every event into a row used by the UI. `TicketSanitizer` subscribes to entity events to compute the sanitized form. `TicketWorkflow` both reads (`getTicket`) and writes (`markAdvising`, `recordRecommendation`, `recordEvaluation`, `fail`) on the entity. The relationship between `ResolutionAdvisorAgent` and `RecommendationSet` is "returns" — the agent's task result is the recommendation record.

## Defence-in-depth governance flow

For any recommendation that lands in the entity log, the ticket passed through:

1. **PII sanitizer** — the model never sees customer identifiers; the audit log retains the raw form.
2. **ResolutionAdvisorAgent** — one model call, one structured output grounded in the resolution library.
3. **On-decision evaluator** — every recommendation gets a 1–5 grounding score so the support agent knows which suggestions to trust at a glance.

Each step is independent. Removing either check opens an explicit gap the other does not silently cover.
