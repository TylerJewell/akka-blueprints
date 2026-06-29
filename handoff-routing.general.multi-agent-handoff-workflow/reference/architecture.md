# Architecture — multi-agent-handoff-workflow

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`TaskSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundTaskReceived` events into `TaskQueue` (event-sourced for audit). An `AdmissionGuardrail` Consumer subscribes to that queue, checks admission criteria, registers a `TaskEntity`, and starts a `HandoffWorkflow` instance. Tasks that fail admission never reach the workflow.

The workflow orchestrates the handoff. It calls `RouterAgent` to classify the task, then branches: `DATA_ANALYSIS` invokes `DataAnalyst`, `CONTENT_WRITING` invokes `ContentWriter`, `CODE_REVIEW` invokes `CodeReviewer`, `UNROUTABLE` terminates in `REJECTED`. Each specialist's tool calls pass through the before-tool-call guardrail embedded in the workflow step. If a tool call is blocked, the task terminates immediately in `TOOL_BLOCKED`. If the specialist returns a `TaskResult`, a validation step checks structural correctness before publishing.

A second Consumer, `RoutingEvalScorer`, runs independently of the workflow. It subscribes to `TaskEntity` events; on every `RoutingDecided` it invokes `RoutingJudge` and writes a `RoutingScored` event back to the same entity. This is the **on-decision eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the data-analysis happy path). Two distinct concurrent flows merge into `TaskEntity`:

1. The workflow path: `routeStep` → `branchStep` → `dataStep` → `validateStep` → `publishStep`.
2. The eval path: `RoutingEvalScorer` observes `RoutingDecided` and writes `RoutingScored` in parallel.

Both write to the same `TaskEntity`; the entity's commands are idempotent on `taskId` and the events do not collide (different fields are mutated). The view materialises both events independently, so the App UI sees a routing score appear within ~10 s of the routing decision regardless of which path completes first.

## State machine

Eleven states. The interesting branches:

- After `RECEIVED`, the admission check produces either `ADMITTED` (workflow starts) or `REJECTED` (terminal — no agent ever activates). This is the earliest possible gate in the system.
- After `ADMITTED`, the routing decision moves the task to `ROUTED`, then to one of three `ROUTED_*` states based on domain. `UNROUTABLE` jumps straight to `REJECTED` — no specialist is invoked.
- After the specialist finishes executing, the before-tool-call guardrail may have already blocked the task in `TOOL_BLOCKED` (terminal). If not, the validation step produces either `COMPLETED` (terminal) or `VALIDATION_FAILED` (terminal).

`RoutingScored` events do not change `status`; they attach the eval score. The state diagram omits this as a no-op for clarity.

## Entity model

`TaskEntity` is the source of truth and emits eleven event types covering receipt, admission, rejection-at-admission, routing, domain routing, tool-call blocking, draft, validation failure, publication, task rejection, and the eval score. `TaskQueue` is the upstream audit log — only `AdmissionGuardrail` subscribes to it. `TaskView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any task result that reaches `COMPLETED`, the task passed through:

1. **AdmissionGuardrail** — structural and content checks before any LLM is activated. Malformed or out-of-scope requests never enter the workflow.
2. **RouterAgent** — typed classifier with a default-to-`UNROUTABLE` policy that prevents quietly mis-routing ambiguous content to the wrong specialist.
3. **Specialist agent** — owns the execution end-to-end with a tightly-scoped prompt (data-analyst cannot invent numbers; content-writer cannot invent citations; code-reviewer cannot execute code).
4. **Before-tool-call guardrail** — intercepts every tool invocation before it runs. Disallowed tool names, argument shapes, or external-network calls are blocked immediately.
5. **Validation step** — structural check on the returned `TaskResult` before it is written as `ResultPublished`.
6. **RoutingEvalScorer** — out-of-band scoring of every routing decision, providing a continuous quality signal.

Steps 1 and 4 are preventive (bad requests and disallowed tool use are blocked before they can produce harm). Step 5 is a structural gate (malformed results are caught before publication). Step 6 is continuous (a sustained drop in routing scores signals a regression).
