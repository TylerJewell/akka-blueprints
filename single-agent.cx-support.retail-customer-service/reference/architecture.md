# Architecture — retail-customer-service

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph centres on one decision-making LLM call. `CustomerServiceEndpoint` accepts a session start or a new turn and calls `SessionEntity` to record the turn, then starts a `SessionWorkflow` instance. The workflow's `agentStep` invokes `CustomerServiceAgent` — the single AutonomousAgent — with the conversation history and product/order context packed as a `TaskDef.attachment("context.json", ...)`. The agent has two guardrails registered on it: `OrderModificationGuardrail` (before-tool-call on order mutation tools) and `ReplyPolicyGuardrail` (before-agent-response on all outbound replies). Once a well-formed `AgentReply` passes both guardrails, the workflow's `applyModificationStep` applies any requested order change to `OrderEntity`, and `replyStep` records the reply on `SessionEntity`. Both entities project their events into read-model views (`SessionView`, `OrderView`) that `CustomerServiceEndpoint` serves over REST and SSE.

The graph has no second agent. `OrderModificationGuardrail` and `ReplyPolicyGuardrail` are logic classes with no LLM calls — one fetches entity state, the other scans text against policy files. That is what makes this a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces J2: a customer requests an address change on a PENDING order. Two distinct decision points occur:

1. The `before-tool-call` guardrail fires before `updateOrderAddress` executes. It fetches live order status from `OrderEntity` — if the order were SHIPPED, the call would be rejected here without the tool ever running.
2. The `before-agent-response` guardrail fires on the candidate reply. In the happy path, the reply passes immediately. On a policy violation (competitor mention, price undercut), the agent loop retries from the previous iteration.

The order mutation (`applyModificationStep`) is a separate workflow step that runs after the agent turn concludes, ensuring the agent's reply and the order change are recorded independently — a modification failure does not suppress the agent's reply.

## State machine — SessionEntity

Five states. The key characteristic is that `ACTIVE` is a looping state: each new turn brings the session through `TurnStarted → TurnReplied` (or `TurnBlockedByGuardrail` / `TurnFailed`) without transitioning the session itself until the customer or the system explicitly closes it. This allows multi-turn conversations within a single session entity.

## State machine — OrderEntity

Five states with four terminal states (`DELIVERED`, `CANCELLED`). The before-tool-call guardrail is the primary enforcement point; the entity's command handlers provide a second-line defense — any direct API call that attempts an invalid modification is rejected at the entity level even without the guardrail path.

## Entity model

`SessionEntity` and `OrderEntity` are independent aggregates. `SessionWorkflow` coordinates writes across both: it reads from `OrderEntity` (via the guardrail's entity lookup), then writes to `OrderEntity` in `applyModificationStep`, then writes to `SessionEntity` in `replyStep`. The two entity event streams project independently into `SessionView` and `OrderView`. There is no cross-entity event subscription — the workflow is the only coordinator.

## Defence-in-depth governance flow

For any order modification that lands on `OrderEntity`, it passed through:

1. **before-tool-call guardrail** — live order status validated before the tool executes; statuses outside the permitted set block the call with a structured code.
2. **OrderEntity command handler** — second-line check: the entity rejects any command that arrives with an invalid status, regardless of whether it came through the agent path.

For any customer-facing reply that reaches `SessionEntity`, it passed through:

3. **before-agent-response guardrail** — competitor names, price undercuts, and rude phrases blocked before the reply leaves the agent loop.

Each layer is independent. Removing any one of them opens a gap the remaining layers do not silently cover.
