# Architecture — guardrails-pattern

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one LLM call protected by two guardrail hooks. `InteractionEndpoint` accepts a prompt submission, writes a `PromptSubmitted` event onto `InteractionEntity`, and starts `InteractionWorkflow`. The workflow's `promptGuardStep` invokes `PromptGuardrail.check()` — if the prompt matches a blocked-topic pattern, the entity transitions to `BLOCKED` and the workflow ends without touching the model. If the prompt passes, the workflow's `agentStep` calls `AssistantAgent` — the single AutonomousAgent — with the prompt as task instructions. Inside the agent, the same `PromptGuardrail` runs again as a `before-llm-call` hook; `ReplyGuardrail` then runs as a `before-agent-response` hook on each candidate reply. Once a reply passes both hooks, it flows back to the workflow, which writes `ReplyRecorded` and then runs a post-hoc `replyGuardStep` to emit the audit event. `InteractionView` projects every entity event into a read-model row; `InteractionEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The post-hoc `replyGuardStep` in the workflow is a direct call to `ReplyGuardrail.check()` — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments shape the timeline:

1. The `promptGuardStep` runs synchronously in the workflow before the agent is invoked. In normal operation this completes in sub-milliseconds — it is a vocabulary check, not a network call.
2. The `agentStep` is bounded by a 60 s timeout to accommodate LLM latency. The `before-llm-call` and `before-agent-response` hooks run inside the agent loop and count against the task's `maxIterationsPerTask(3)` budget when they reject.

The `replyGuardStep` is a record-keeping step only — it runs the same check the hook already ran inside the agent, so it is deterministic and fast.

## State machine

Six states. The notable paths:

- **Happy path**: `SUBMITTED → PROMPT_CHECKED → REPLYING → REPLY_RECORDED`.
- **Blocked path**: `SUBMITTED → BLOCKED`. This is the key path the blueprint demonstrates: the model is never called.
- **Failure path**: `REPLYING → FAILED`. This covers guardrail-exhaustion (all three retries failed validation) and agent errors.
- `BLOCKED` and `REPLY_RECORDED` are both terminal states — the interaction is complete.
- There is no `APPROVED` or `REJECTED` state in the human-oversight sense. Replies surface directly to the user; human review is optional and not modeled by the entity.

## Entity model

`InteractionEntity` is the source of truth. It emits seven event types, two of which (`PromptGuardChecked` and `ReplyGuardChecked`) carry a `GuardrailOutcome` payload used by the UI timeline. `InteractionView` projects every event. `InteractionWorkflow` both reads (`getInteraction`) and writes (`recordPromptGuardResult`, `recordReply`, `recordReplyGuardResult`, `fail`) on the entity. `AssistantAgent`'s task result is the `AgentReply` record; `PromptGuardrail` and `ReplyGuardrail` produce `GuardrailOutcome` records.

## Two-sided guardrail coverage

For any reply that lands in the entity log, the prompt passed through:

1. **PromptGuardrail (workflow pre-check)** — synchronous vocabulary check before the agent call; entity records the result.
2. **PromptGuardrail (before-llm-call hook)** — same check, registered on the agent; fires inside the agent loop so the model call is blocked at the boundary if somehow bypassed at the workflow layer.
3. **AssistantAgent** — one model call, one structured output.
4. **ReplyGuardrail (before-agent-response hook)** — structural validation and topic re-check on every candidate reply; forces retry on failure.
5. **ReplyGuardrail (workflow post-check)** — deterministic re-check to emit the audit event; does not add a new gate but makes the outcome durable on the entity.

The two-hook pattern means that neither the input guardrail nor the output guardrail is a single point of failure. A bypassed input check is still caught inside the agent by the `before-llm-call` hook; a policy-compliant input that generates a non-compliant reply is caught by the `before-agent-response` hook.
