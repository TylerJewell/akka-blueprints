# Architecture — akka-async-agent-mesh

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`MeshEndpoint` is the entry point. A dispatch request is logged as a `DispatchRequestSubmitted` event on `MessageBusEntity` (event-sourced for audit). `MessageBusConsumer` subscribes to that entity and starts a `MessageDispatchWorkflow` per submission. The workflow runs `DispatcherAgent`, which composes a `MessagePayload` and calls the `send_message_to_agent_async` tool. Every tool call passes the G1 before-tool-call guardrail; if the recipient is not in the allowed roster or the mesh is halted, the send is refused and the request is recorded `SEND_BLOCKED`. On a successful send, the workflow creates a `MessageEntity` (status `DISPATCHED`), records a `DeliveryReceiptEntity`, and terminates — the dispatcher does not wait.

A separate `MessageBusConsumer` subscription (or the same consumer with a second handler) picks up the `MessageDispatched` event and starts a `MessageProcessingWorkflow` for that message. The workflow runs `PiiSanitizer` on the payload body (S1), loads the recipient agent's `AgentMemoryEntity` context, runs `ProcessorAgent`, writes new memory entries back to `AgentMemoryEntity`, and marks the `MessageEntity` `PROCESSED`. If the processor emits a `followUp`, the workflow submits a new `DispatchRequest` to `MessageBusEntity`, beginning another dispatch cycle. `MessageBusView` projects every `MessageEntity` transition into a shared list the App UI streams via SSE.

Two TimedActions run alongside: `ScenarioSimulator` drips sample scenarios; `StaleMessageMonitor` flags orphaned `DISPATCHED` messages.

## Interaction sequence

The sequence diagram traces the happy path (J1): submit a dispatch request → dispatcher composes and sends (guardrail gate marked) → `MessageEntity` created → processor receives, sanitizes, and processes → memory updated → message reaches `PROCESSED`. The diagram makes the async boundary explicit: `MessageDispatchWorkflow` terminates before `MessageProcessingWorkflow` starts. Both workflows run on separate timelines, coordinated only through `MessageEntity` events.

## State machine

`MessageEntity` tracks the full lifecycle. A message begins at `DISPATCHED` when the dispatch workflow creates it. The workflow immediately transitions it to `DELIVERED` once the receipt is recorded. `MessageProcessingWorkflow` moves it to `PROCESSING` when it starts, and to `PROCESSED` when the processor's result is written. Two terminal states branch off `DISPATCHED`: `SEND_BLOCKED` when the guardrail refuses the send (no `MessageEntity` advances past this), and `STALE` when `StaleMessageMonitor` detects a message that has not been delivered within the timeout. Both `SEND_BLOCKED` and `STALE` are terminal — they indicate the message will not be processed.

## Entity model

`MessageEntity` is the source of truth for each in-flight message and projects into `MessageBusView` — the only read-side surface the UI and the processing workflow read. `AgentMemoryEntity` is keyed by agent id and holds the growing set of `MemoryEntry` facts an agent accumulates across exchanges; each agent instance owns its own entity so memory stays isolated. `DeliveryReceiptEntity` captures the tool's acknowledgement separately so it can be queried independently of the message lifecycle. `MessageBusEntity` is the audit log — every `DispatchRequest` is recorded there regardless of outcome.

## Concurrency & timeouts

- Fire-and-forget is enforced structurally: `MessageDispatchWorkflow` ends after writing the receipt; it has no reference to `MessageProcessingWorkflow` and cannot block on it.
- Per-step timeout: 90 s on the agent-calling steps (`MessageDispatchWorkflow.composeStep`, `MessageProcessingWorkflow.processStep`).
- PII sanitizer (`PiiSanitizer`) is a deterministic pure function — not an LLM call — so sanitization is fast, predictable, and reproducible across retries.
- The G1 guardrail reads `async-mesh.agents` from `application.conf` at startup; the roster check is a constant-time map lookup with no entity access, so it does not add latency to the happy path.
- `StaleMessageMonitor` fires every 90 s and advances messages that have been `DISPATCHED` for more than 3 minutes to `STALE`, keeping the board free of orphaned rows from failed workflows.
- Memory isolation: `AgentMemoryEntity("agent-alpha")` and `AgentMemoryEntity("agent-beta")` are separate entity instances with separate event journals. A write to one never affects the other.
