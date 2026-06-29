# Architecture — cross-agency-case-handoff-mesh

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`CaseSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundCaseReceived` events into `CaseInbox` (event-sourced for audit). A `PiiScopingFilter` Consumer subscribes to that inbox, scopes the payload to the receiving agency's authorised field set, registers a `CaseEntity`, and starts a `HandoffWorkflow` instance.

The workflow orchestrates the mesh. It calls `JurisdictionRouter` to classify the scoped case, then enters a loop: before invoking any agency agent, it calls `JurisdictionGuardrail` to verify the receiving agency's jurisdiction. If jurisdiction is cleared, it invokes the assigned agent (`IntakeAssessor` or `BenefitsReviewer`) and waits for a typed `SegmentOutcome`. After the agent completes, the workflow enters the HITL approval step — it emits `HandoffPending` and suspends until an operator calls `approve-handoff` or `reject-handoff` on the `CaseEndpoint`. On approval, if a further segment exists the loop restarts; if not, the case is closed. On rejection, the workflow terminates in `HANDOFF_REJECTED`.

There is no background eval consumer in this blueprint. The governance signal comes from the HITL record (who approved the handoff, when, with what note) and from the jurisdiction guardrail violation log.

## Interaction sequence

The sequence diagram traces J1 (the intake-to-benefits happy path with operator approval). The HITL suspension (steps 14–16) is the most distinctive part of this pattern: the workflow blocks until the operator acts. No downstream processing can occur while the case sits in `HANDOFF_PENDING`.

The `JurisdictionGuardrail` fires at step 8, before `IntakeAssessor` is invoked at step 10. This ordering is strict — the guardrail always runs first in each iteration of the route → guard → agent → approval loop.

## State machine

Twelve states. The key branches:

- After `ROUTED`, the `jurisdiction check` step runs. `jurisdiction.allowed=true` → `IN_SEGMENT` (agent invoked). `jurisdiction.denied` → `JURISDICTION_BLOCKED` (terminal). This is the before-agent-invocation guardrail made visible in the state machine.
- After `SEGMENT_COMPLETE`, the case enters `HANDOFF_PENDING` for every completed segment, regardless of whether there is a next segment. The operator's approval determines whether the workflow continues or closes.
- `HANDOFF_EMITTED` → `CLOSED` is the two-segment terminal path. A single-segment case goes directly from `HANDOFF_EMITTED` to `CLOSED` in the same approval action.
- `UNROUTABLE` and `JURISDICTION_BLOCKED` are both reachable without any agency agent ever being invoked.

## Entity model

`CaseEntity` is the source of truth and emits twelve event types covering every state transition, including the HITL approval and rejection events. `CaseInbox` is the upstream audit log — only `PiiScopingFilter` subscribes to it. `CaseView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any case determination that reaches the operator's approval panel, it has passed through:

1. **PII scoping filter** — the agency agent never sees fields outside its authorised scope. Dropped field categories are logged.
2. **JurisdictionRouter** — typed classifier that defaults to `UNROUTABLE` under ambiguity, preventing unrecognised cases from entering any agency segment.
3. **JurisdictionGuardrail** — before-agent-invocation check against the receiving agency's declared jurisdiction scope. A mismatch blocks the invocation entirely; the agent is never called.
4. **Agency agent (IntakeAssessor or BenefitsReviewer)** — autonomous agent with a tightly scoped prompt: no invented thresholds, no benefit amounts, no legal advice, escalate when uncertain.
5. **HITL approval step** — the case-owner at the current agency reviews the automated determination and explicitly approves or rejects the handoff. No determination crosses an agency boundary without a human sign-off.

Step 1 is preventive (the model cannot disclose what it never receives). Step 3 is preventive (the model is never invoked outside scope). Step 5 is the human gate that enforces accountability for every inter-agency transfer.
