# Architecture — akka-router-pattern

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`TaskSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundTaskReceived` events into `TaskQueue` (event-sourced for audit). A `QueueConsumer` Consumer subscribes to that queue, registers a `RequestEntity`, and starts a `RouterWorkflow` instance.

The workflow orchestrates the routing. It calls `ClassifierAgent` to classify the request, then immediately calls `RoutingGuardrail` with the classification in hand — this is the `before-agent-invocation` hook. If the guardrail denies the request, the workflow terminates in `BLOCKED` before any specialist is selected. If the guardrail allows, the workflow branches: `CONTENT` invokes `ContentSpecialist`, `CODE` invokes `CodeSpecialist`, `DATA` invokes `DataSpecialist`, `UNKNOWN` terminates in `UNROUTABLE`. The chosen specialist owns the `EXECUTE` task end-to-end and returns a typed `TaskResult`, which the workflow then publishes.

A second Consumer, `RoutingEvalScorer`, runs independently of the workflow. It subscribes to `RequestEntity` events; on every `ClassificationDecided` it invokes `RoutingJudge` and writes a `RoutingScored` event back to the same entity. This is the **on-decision eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the content happy path). Two distinct concurrent flows merge into `RequestEntity`:

1. The workflow path: `classifyStep` → `guardrailStep` → `routeStep` → `contentStep` → `publishStep`.
2. The eval path: `RoutingEvalScorer` observes `ClassificationDecided` and writes `RoutingScored` in parallel.

Both write to the same `RequestEntity`; the entity's commands are idempotent on `requestId`. The view materialises both events independently, so the App UI sees a routing score appear within ~10 s of the classification decision regardless of which path completes first.

## State machine

Ten states. The key branching points:

- After `CLASSIFIED`, the guardrail verdict branches the flow. `ALLOWED` moves to `GUARDRAIL_PASSED`; `DENIED` jumps directly to the `BLOCKED` terminal — no specialist is reached. `UNKNOWN` domain also short-circuits here to `UNROUTABLE`.
- After `GUARDRAIL_PASSED`, the domain value branches the flow. `CONTENT`, `CODE`, and `DATA` go to their respective `ROUTED_*` states, then converge at `EXECUTING` once the specialist begins. After the specialist finishes, `ResultPublished` transitions to the `COMPLETED` terminal.
- From `BLOCKED`, the only forward transition is operator-initiated unblock → `COMPLETED`. Blocked requests wait indefinitely.

`RoutingScored` events do not change `status`; they attach the eval score to the entity. The state diagram omits this as a no-op for clarity.

## Entity model

`RequestEntity` is the source of truth and emits nine event types covering registration, classification, guardrail verdict, routing, execution start, block, result, unroutable, and the eval score. `TaskQueue` is the upstream audit log — only `QueueConsumer` subscribes to it. `RequestView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any result that goes out, the request passed through:

1. **ClassifierAgent** — typed classifier with a default-to-`UNKNOWN` policy that prevents quietly mis-routing ambiguous content.
2. **RoutingGuardrail** — typed before-agent-invocation check. Blocks prompt-injection tokens, out-of-scope data access requests, cross-domain confusion, and loop-detection patterns before any specialist begins.
3. **Specialist agent** — owns the execution end-to-end with a tightly-scoped prompt (no invented facts, explicit escalation for out-of-scope requests).
4. **RoutingEvalScorer** — out-of-band scoring of every classification decision.

Step 2 is preventive (no specialist ever sees a blocked payload). Step 4 is continuous (a sustained drop in routing scores would signal a regression).
