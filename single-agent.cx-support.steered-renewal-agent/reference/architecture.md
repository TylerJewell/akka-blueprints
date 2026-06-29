# Architecture — steered-renewal-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `RenewalEndpoint` accepts a renewal request, writes a `RenewalRequested` event onto `LoanEntity`, and immediately starts a `RenewalWorkflow` instance. The workflow's `enrichStep` loads the patron record and loan detail from in-process seed data and writes them back via `attachEnrichedData`. The `decideStep` calls `RenewalAgent` — the single AutonomousAgent — with the patron and loan context formatted as task instructions. Before any candidate decision leaves the agent loop, `PolicyEnforcer` validates it against four hard policy rules. A decision that passes all checks flows to `decideStep`'s success path, which calls `LoanEntity.recordDecision`. The `notifyStep` writes a `NotificationRecord` in-process and transitions the entity to COMPLETED. `RenewalView` projects every entity event into a read-model row; `RenewalEndpoint` serves the read model to the UI over REST and SSE.

There is no second agent. The notification step is a deterministic in-process record write, not an LLM call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). There are two distinct actor hand-offs:

1. `RenewalEndpoint` writes the initial event and starts the workflow; the workflow owns the remaining lifecycle from there.
2. The workflow calls `RenewalAgent` via `runSingleTask`; the agent's loop consults `PolicyEnforcer` before returning. The workflow does not receive a decision until the guardrail passes it.

The LLM call is bounded by `decideStep`'s 60 s timeout. `enrichStep` and `notifyStep` are bounded at 10 s and 5 s respectively — they are in-process lookups and record writes with no external latency.

## State machine

Six states. The interesting transitions:

- The happy path is `REQUESTED → ENRICHED → DECIDING → DECISION_RECORDED → COMPLETED`.
- Two failure transitions land in `FAILED`: an enrich error during `REQUESTED`, and an agent error (or guardrail-exhaustion after three iterations) during `DECIDING`. A `FAILED` renewal retains all prior data on the entity — the UI shows the partial state for administrator review.
- There is no `APPEALED` or `MANUALLY_OVERRIDDEN` state. The decision is enforced by the system; any override is an out-of-band library-staff action not modelled in this blueprint.

## Entity model

`LoanEntity` is the source of truth. It emits six event types. `RenewalView` projects every event into a row used by the UI. `RenewalWorkflow` both reads (`getRenewal`) and writes (`attachEnrichedData`, `markDeciding`, `recordDecision`, `recordNotification`, `fail`) on the entity. The relationship between `RenewalAgent` and `RenewalDecision` is "returns" — the agent's task result is the decision record.

## Steering governance flow

The guardrail provides a hard enforcement layer at the agent boundary:

1. **RenewalAgent** receives patron and loan context; produces a candidate `RenewalDecision`.
2. **PolicyEnforcer (before-agent-response)** — checks fine threshold, renewal count, due-date window, and outcome enum validity. A violating candidate is rejected; the agent loop retries within its 3-iteration budget. A patron's account is never updated with a policy-violating decision.
3. **LoanEntity** — the first decision that reaches it via `recordDecision` is guaranteed to have passed all four policy checks.

Removing the guardrail would allow the agent to approve renewals that violate library policy, with no other component intercepting the violation before the entity event is written.
