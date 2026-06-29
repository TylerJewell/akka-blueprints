# Architecture — incident-classifier

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ClassificationEndpoint` accepts an incident submission and writes an `IncidentSubmitted` event onto `IncidentEntity`. The `VocabularyValidator` Consumer subscribes, counts the loaded taxonomy categories and CI entries, and writes a `TaxonomyScope` back via `markValidated`. The same Consumer then starts a `ClassificationWorkflow` instance. The workflow's `classifyStep` calls `IncidentClassifierAgent` — the single AutonomousAgent — with the closed taxonomy as `TaskDef.instructions(...)` and the incident description fields as a `TaskDef.attachment(...)`. Once the agent returns a `ClassificationResult`, the workflow writes `ClassificationRecorded` and runs `AccuracyEvaluator` in `evalStep`. The score lands as `AccuracyScored`. `ClassificationView` projects every entity event into a read-model row; `ClassificationEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `AccuracyEvaluator` is a deterministic rule-based scorer — it reads the `TaxonomyTable` singleton and computes arithmetic. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct pauses occur:

1. The `VocabularyValidator` subscription lag between `IncidentSubmitted` and `TaxonomyValidated` — sub-second in normal operation because the taxonomy is loaded in-process at startup.
2. The `awaitValidatedStep` polling loop inside the workflow — polls `IncidentEntity` every 1 s up to its 15 s timeout, advancing as soon as `incident.scope().isPresent()` returns true.

The agent call is bounded by `classifyStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — the taxonomy lookup is an in-memory Map, and the scoring arithmetic involves no I/O.

## State machine

Six states. Key paths:

- The happy path is `SUBMITTED → TAXONOMY_VALIDATED → CLASSIFYING → CLASSIFICATION_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a taxonomy-loader error during `SUBMITTED`, and an agent error (or iteration-budget exhaustion) during `CLASSIFYING`. A `FAILED` incident retains its prior data on the entity for manual inspection.
- There is no `ROUTED` or `RESOLVED` state. The classification is an input to the downstream routing workflow; this blueprint stops at `EVALUATED`. The routing step is the deployer's responsibility.

## Entity model

`IncidentEntity` is the source of truth. It emits six event types. `ClassificationView` projects every event into a row consumed by the UI and the accuracy-window query. `VocabularyValidator` subscribes to entity events to trigger taxonomy validation and workflow startup. `ClassificationWorkflow` both reads (`getIncident`) and writes (`markClassifying`, `recordClassification`, `recordAccuracy`, `fail`) on the entity. The relationship between `IncidentClassifierAgent` and `ClassificationResult` is "returns" — the agent's task result is the classification record.

## Governance flow

For any classification that lands in the entity log, the incident passed through:

1. **VocabularyValidator** — the taxonomy is confirmed loaded and the scope is recorded before the agent is invoked.
2. **IncidentClassifierAgent** — one model call, one structured output with category, subcategory, and CI drawn from a closed taxonomy.
3. **AccuracyEvaluator (on-decision)** — every classification is immediately scored: does the category exist? does the subcategory belong? does the CI appear in the registry? Score ≤ 2 flags the card for human review.
4. **ClassificationView (continuous-accuracy)** — the rolling 7-day window aggregates all accuracy contributions so ops managers can detect drift before it affects SLAs.

Each layer is independent. A category code that slips past the agent still shows up in the evaluator. A temporary model degradation shows up in the rolling window. Neither layer relies on the other to catch the same defect.
