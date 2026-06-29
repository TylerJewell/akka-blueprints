# Architecture — basic-cv-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one LLM call. `CvEndpoint` accepts a profile submission and writes a `ProfileSubmitted` event onto `CvEntity`. The `ProfileSanitizer` Consumer subscribes, redacts PII from the contact email and any free-text additional notes, and writes the sanitized profile back via `attachSanitized`. The same Consumer then starts a `CvWorkflow` instance. The workflow's `generateStep` calls `CvGeneratorAgent` — the single AutonomousAgent — with output-mode instructions as `TaskDef.instructions(...)` and the sanitized profile JSON as a `TaskDef.attachment(...)`. Once the agent returns a `GeneratedCv`, the workflow validates that the headline and sections are non-empty before writing `CvGenerated` to the entity. `CvView` projects every entity event into a read-model row; `CvEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. There is no guardrail component because structural validation of the agent's output (non-empty headline and sections) is handled by the workflow step itself before the result lands on the entity. This makes the blueprint a minimal **single-agent** example: one component talks to a model, and the single governance mechanism (the PII sanitizer) runs before that call.

## Interaction sequence

The sequence traces the happy path (J1). Two moments where the system waits:

1. The `ProfileSanitizer` subscription lag between `ProfileSubmitted` and `ProfileSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `CvEntity` every 1 s up to its 15 s timeout, advancing as soon as `requestState.sanitized().isPresent()` returns true.

The agent call itself is bounded by `generateStep`'s 60 s timeout. The workflow's inline validation (non-empty output check) runs in milliseconds — no external service, no LLM call.

## State machine

Five states. The interesting paths:

- The happy path is `SUBMITTED → SANITIZED → GENERATING → CV_GENERATED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error or empty-output validation failure during `GENERATING`. A `FAILED` request's prior data is preserved on the entity — the UI shows the partial state for the recruiter.
- There is no `APPROVED` or `SENT` state. The generated CV is a draft; the recruiter reviews it before forwarding. The blueprint deliberately stops at `CV_GENERATED`.

## Entity model

`CvEntity` is the source of truth. It emits five event types. `CvView` projects every event into a row used by the UI. `ProfileSanitizer` subscribes to entity events to compute the sanitized form. `CvWorkflow` both reads (`getRequest`) and writes (`markGenerating`, `recordGenerated`, `fail`) on the entity. The relationship between `CvGeneratorAgent` and `GeneratedCv` is "returns" — the agent's task result is the generated CV record.

## PII governance flow

For any CV that lands in the entity log, the profile data passed through:

1. **PII sanitizer** — the model never sees the raw contact email, government identifiers, financial account data, or sensitive free-text; the audit log retains the raw form on the entity.
2. **CvGeneratorAgent** — one model call, one structured output (`GeneratedCv`).
3. **Workflow validation** — the workflow confirms the returned CV has a non-empty headline and at least one section before committing it; empty outputs fail the request rather than silently landing.

Each step is independent. Removing the sanitizer would expose identifiers to the model without any other mechanism compensating.
