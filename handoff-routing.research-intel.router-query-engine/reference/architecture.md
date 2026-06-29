# Architecture — router-query-engine

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`QuestionSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundQuestionReceived` events into `QuestionQueue` (event-sourced for audit). A `QuestionConsumer` Consumer subscribes to that queue, registers a `QueryEntity` for each incoming question, and starts a `QueryWorkflow` instance.

The workflow orchestrates the handoff. It calls `RouterAgent` to classify the question, then branches: `STRUCTURED` invokes `StructuredDataEngine`, `SEMANTIC` invokes `SemanticSearchEngine`, `UNCLEAR` terminates in `ESCALATED`. The chosen engine owns the `ANSWER` task end-to-end and returns a typed `Answer`. The workflow then publishes the answer directly — there is no guardrail step in this blueprint, because research-intelligence answers are advisory and the routing eval provides the continuous quality signal.

A second Consumer, `RoutingEvalScorer`, runs independently of the workflow. It subscribes to `QueryEntity` events; on every `RoutingDecided` it invokes `RoutingJudge` and writes a `RoutingScored` event back to the same entity. This is the **on-decision eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the structured-data happy path). Two distinct concurrent flows merge into `QueryEntity`:

1. The workflow path: `routeStep` → `branchStep` → `structuredStep` → `publishStep`.
2. The eval path: `RoutingEvalScorer` observes `RoutingDecided` and writes `RoutingScored` in parallel.

Both write to the same `QueryEntity`; the entity's commands are idempotent on `queryId` and the events do not collide (different fields are mutated). The view materialises both events independently, so the App UI sees a routing score appear within ~10 s of the routing decision regardless of which path completes first.

## State machine

Seven states. The interesting branches:

- After the `ROUTING` transition, the `engineType` value branches the flow. `STRUCTURED` and `SEMANTIC` go to their respective `ROUTED_*` state, then both converge at `ANSWERING` once the engine returns. `UNCLEAR` jumps straight to the `ESCALATED` terminal — no engine is invoked at all.
- After `ANSWERING`, the workflow publishes and the status moves to `ANSWERED` (terminal). There is no blocked state in this pattern — the on-decision eval is the quality signal rather than a blocking guardrail.

`RoutingScored` events do not change `status`; they attach the eval score to the entity. The state diagram omits this as a no-op for clarity.

## Entity model

`QueryEntity` is the source of truth and emits seven event types covering registration, routing, routed acknowledgement, answer draft, publish, escalate, and the eval score. `QuestionQueue` is the upstream audit log — only `QuestionConsumer` subscribes to it. `QueryView` projects the entity into a single read-model row.

## Governance flow

For any answer that is published, the query passed through:

1. **RouterAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous questions.
2. **Engine agent** — owns the answer end-to-end with a tightly-scoped prompt (no fabricated figures, cite real sources, escalate when data is insufficient).
3. **RoutingEvalScorer** — out-of-band scoring of every routing decision against the question's actual information need.

Step 1 is preventive (the wrong engine never sees the question). Step 3 is continuous (a sustained drop in routing scores would signal router-prompt regression or taxonomy drift).
