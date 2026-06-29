# Architecture — per-tenant-agent-fleet

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`FleetEndpoint` is the entry point. A customer registration is logged as a `CustomerRegistered` event on `CustomerRegistry` (event-sourced for audit and replay). `CustomerIngestConsumer` subscribes to that registry, initializes an `AgentMemoryEntity`, and starts an `AgentFleetWorkflow` keyed by the customer id. That workflow is the per-customer pipeline: it runs through four stages, persisting each result onto the shared entities before moving on.

Three coordination capabilities operate under the same pipeline:

- **Control-plane delegation.** `registerStep` invokes `FleetCoordinator` to record the fleet entry. `researchStep` fans out one `BackgroundResearcher` instance per customer (addressed by customer id) to build a `CustomerProfile`, then stores it in `AgentMemoryEntity`.
- **Outbound tool dispatch with guardrail.** `welcomeStep` calls `CustomerTools.sendWelcomeEmail` — a single guarded tool call that the before-tool-call guardrail vets for recipient domain, body size, and template restrictions. A refused call records the block reason and lets the workflow proceed.
- **Ongoing chat over shared memory.** After `readyStep`, each incoming `POST /api/customers/{id}/chat` resumes the same `AgentFleetWorkflow` for that customer via `chatStep`, which runs `ChatResponder` with the customer's full `AgentMemoryEntity` as context and appends the new turn when done.

`FleetEvalConsumer` subscribes to memory events and records a non-blocking quality eval after each welcome and chat turn. `StaleAgentMonitor` scans the `FleetStatusView` every 5 minutes and builds a `FleetHealthSummary` list for dormant customers. `AppEndpoint` serves the embedded UI and the metadata the tabs read.

## Interaction sequence

The sequence diagram traces the happy path (J1): register → sanitize → coordinator records fleet entry → background researcher builds profile → welcome email dispatched through the guardrail → agent enters READY → customer sends a chat message → responder draws on memory and replies. A `Note over` block marks the PII sanitizer firing at the boundary before any agent sees the payload. A second marks where the before-tool-call guardrail vets the email dispatch. A third marks the async eval consumer recording the welcome stage score.

## State machine

`CustomerRegistry` is the lifecycle spine. It moves `REGISTERED → ONBOARDING → READY`. The `READY` state is a steady state: each chat turn arrives as a resumed workflow step but does not change the customer's registry status — it appends a `ChatTurn` to `AgentMemoryEntity` instead. There is no terminal state for a customer who is actively engaged; `DORMANT` is a fleet concept managed by `StaleAgentMonitor` at the eval layer, not a `CustomerRegistry` status.

## Entity model

`CustomerRegistry` is the master customer record and the ingest trigger; `CustomerIngestConsumer` subscribes to its events to start the workflow. `AgentMemoryEntity` is the per-customer interaction store: it accumulates the profile, the welcome result, chat turns, and eval results emitted by `FleetEvalConsumer`. `FleetStatusView` projects from both entities into a unified fleet row. `ChatHistoryView` projects `AgentMemoryEntity`'s chat-turn events into an ordered conversation list the UI streams per customer.

## Concurrency and timeouts

- One `AgentFleetWorkflow` per customer, keyed by `customerId`. Two registrations for different customers run in parallel; a second registration for the same customer resumes the same workflow rather than starting a duplicate.
- `ChatResponder` is one instance per customer (same key). Chat turns for the same customer are serialised through the workflow; different customers process in parallel.
- `PiiSanitizer` is synchronous and runs at the service boundary before state is written. No raw PII reaches entity state, agent context, or SSE events.
- Per-step timeouts: `registerStep` 30 s, `researchStep` 90 s, `welcomeStep` 60 s, `chatStep` 60 s. The default 5 s timeout would expire mid-LLM-call.
- A guardrail refusal in `welcomeStep` does not abort the pipeline; the customer's agent still reaches `READY` and can answer chat.
- `FleetEvalConsumer` is downstream and non-blocking; it never gates the workflow.
- `StaleAgentMonitor` is read-only; it never modifies customer or agent state.
