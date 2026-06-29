# Architecture — financial-advisor-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `AdvisoryEndpoint` accepts a client submission and writes an `AdvisoryRequested` event onto `AdvisoryEntity`. The `ClientDataSanitizer` Consumer subscribes, strips regulated financial identifiers (account numbers, tax IDs, national IDs), and writes the sanitized profile back via `attachSanitized`. The same Consumer then starts an `AdvisoryWorkflow` instance. The workflow's `adviseStep` calls `FinancialAdvisorAgent` — the single AutonomousAgent — with the client's question and account type as `TaskDef.instructions(...)` and the sanitized portfolio JSON as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`RecommendationGuardrail`) validates each candidate response. Once a response passes, the workflow writes `ResponseRecorded` and runs the `auditStep` — a deterministic SHA-256 digest computation — which appends `AuditLogged` to the entity. `AdvisoryView` projects every entity event into a read-model row; `AdvisoryEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The audit step is purely deterministic — no model call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct pauses occur:

1. The `ClientDataSanitizer` subscription lag between `AdvisoryRequested` and `ClientDataSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `AdvisoryEntity` every 1 s up to its 15 s timeout, advancing as soon as `advisory.sanitized().isPresent()` returns true.

The agent call is bounded by `adviseStep`'s 60 s timeout. The `auditStep` computes a SHA-256 digest in-process and finishes in milliseconds.

## State machine

Six states. The key paths:

- The happy path is `REQUESTED → SANITIZED → ADVISING → RESPONSE_RECORDED → AUDITED`.
- Two failure transitions reach `FAILED`: a sanitizer error during `REQUESTED`, and an agent error (or guardrail-exhaustion after 3 iterations) during `ADVISING`. A `FAILED` advisory's prior data is preserved on the entity — the UI shows the partial state.
- There is no `APPROVED` or `EXECUTED` state. The advisory is advisory-only; the client or human advisor decides whether to act. The blueprint deliberately stops at `AUDITED`.

## Entity model

`AdvisoryEntity` is the source of truth. It emits six event types. `AdvisoryView` projects every event into a row used by the UI. `ClientDataSanitizer` subscribes to entity events to compute the sanitized form. `AdvisoryWorkflow` both reads (`getAdvisory`) and writes (`markAdvising`, `recordResponse`, `recordAudit`, `fail`) on the entity. The relationship between `FinancialAdvisorAgent` and `AdvisoryResponse` is "returns" — the agent's task result is the advisory record.

## Defence-in-depth governance flow

For any advisory that lands in the entity log, the client data passed through:

1. **Sector sanitizer** — the model never sees account numbers, tax IDs, or national IDs; the audit log retains the raw profile for compliance purposes.
2. **FinancialAdvisorAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — unauthorized asset classes, out-of-range weights, and malformed JSON are caught before the response leaves the agent loop.
4. **Deterministic audit step** — every well-formed response is permanently fingerprinted with a SHA-256 digest so the advisory record cannot be silently modified post-hoc.

Each layer is independent. Removing one opens an explicit gap the others do not cover.
