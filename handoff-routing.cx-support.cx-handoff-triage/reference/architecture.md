# Architecture — cx-handoff-triage

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`ConversationSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundMessageReceived` events into `ConversationQueue` (event-sourced for audit). A `PiiSanitizer` Consumer subscribes to that queue, redacts the payload, registers a `ConversationEntity`, and starts a `ConversationWorkflow` instance.

The workflow orchestrates the handoff. It calls `TriageAgent` to classify the sanitized message, then branches: `SALES` invokes `SalesSpecialist`, `ISSUES_REPAIRS` invokes `IssuesRepairsSpecialist`, `UNCLEAR` terminates in `ESCALATED`. The chosen specialist owns the `RESOLVE` task end-to-end. Within the task loop the specialist may invoke tools; each tool call passes through `ToolCallGuardrail` before execution — a denied call is cancelled and the specialist receives a policy-rejection signal. Once the specialist returns a typed `SpecialistResponse`, the workflow calls `ResponseGuardrail` to check the draft. On `allowed=true`, the workflow publishes; on `allowed=false`, it transitions to `BLOCKED` and the conversation waits until an operator unblocks via `POST /api/conversations/{id}/unblock`.

## Interaction sequence

The sequence diagram traces J1 (the sales happy path with a passing tool call). Three distinct interactions compose into `ConversationEntity`:

1. The inbound path: `ConversationSimulator` → `ConversationQueue` → `PiiSanitizer` → `ConversationEntity.registerIncoming` + `attachSanitized`.
2. The workflow path: `triageStep` → `routeStep` → `salesStep` (with `ToolCallGuardrail` inline) → `guardrailStep` → `publishStep`.
3. Entity projection: every event emitted by the workflow flows through `ConversationEntity` → `ConversationView` → SSE to the browser.

The tool-call guardrail intercept (steps 10–11 in the sequence diagram) is synchronous within the specialist's task loop. If denied, the specialist receives the rejection before any tool side-effect occurs.

## State machine

Ten states. The notable paths:

- After `TRIAGED`, the `category` value branches. `SALES` and `ISSUES_REPAIRS` go to their respective `ROUTED_*` state; `UNCLEAR` goes directly to the `ESCALATED` terminal — no specialist is invoked.
- `TOOL_CALL_BLOCKED` is a transient annotation state reachable from either `ROUTED_SALES` or `ROUTED_ISSUES_REPAIRS`. It does not block the specialist from continuing — the specialist receives a rejection and can use its remaining iteration budget to choose an allowed action. The conversation eventually still reaches `RESPONSE_DRAFTED`.
- After `RESPONSE_DRAFTED`, the response guardrail verdict branches again. `allowed=true` → `RESOLVED` terminal. `allowed=false` → `BLOCKED`. From `BLOCKED`, the only forward transition is operator-initiated unblock → `RESOLVED`. There is no auto-timeout — blocked conversations wait indefinitely.

## Entity model

`ConversationEntity` is the source of truth and emits ten event types covering registration, sanitization, triage, routing, draft, tool-call block, guardrail verdict, publish, block, and escalation. `ConversationQueue` is the upstream audit log — only `PiiSanitizer` subscribes to it. `ConversationView` projects the entity into a single read-model row per conversation.

## Defence-in-depth governance flow

For any reply that reaches a customer, the conversation passed through:

1. **PII sanitizer** — the model never sees raw contact details or order ids.
2. **TriageAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous messages.
3. **Specialist agent** — owns the resolution end-to-end with a tightly-scoped prompt (discount ceiling, no invented delivery windows, no fabricated specifications).
4. **ToolCallGuardrail** — intercepts every side-effecting tool call before execution and cancels calls that exceed policy parameters.
5. **ResponseGuardrail** — typed before-agent-response check against the policy rubric. Blocks invented timelines, out-of-authority discounts, scope violations, and redaction-token echoes.

Steps 1 and 4 are preventive (the model can't leak what it never sees; the tool can't execute what the guardrail cancels). Step 5 is detective (drafts that violate get held). Steps 2 and 3 scope the risk surface by narrowing what each agent is asked to do.
