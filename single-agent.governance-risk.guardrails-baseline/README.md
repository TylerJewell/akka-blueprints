# Akka Sample: Guardrails Baseline

A single content-moderation agent processes a user message and produces a structured moderation decision. Three governance controls wrap the agent: a PII sanitizer that strips identifiers before the message reaches the model, a `before-agent-invocation` guardrail that enforces tool-policy rules, and an `after-llm-response` guardrail that validates the decision's output structure before it leaves the agent loop.

Demonstrates the **single-agent** coordination pattern in the governance-risk domain. The surrounding controls are not decorative — each one intercepts a real failure mode: leaking identifiers to the model, invoking out-of-policy tools, and propagating malformed decisions to downstream systems.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the message corpus lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.governance-risk.guardrails-baseline  ~/my-projects/guardrails-baseline
cd ~/my-projects/guardrails-baseline
```

(Optional) Edit `SPEC.md` to adjust the seeded message set or the tool-policy allowlist (e.g., switch from a generic content-moderation policy to a domain-specific financial-advice or healthcare-triage policy).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ModerationAgent** — an AutonomousAgent that accepts a message plus a policy set and returns a typed `ModerationDecision`.
- **ModerationWorkflow** — orchestrates sanitize-wait → moderate → audit per submitted message.
- **ModerationEntity** — an EventSourcedEntity holding the per-moderation lifecycle.
- **MessageSanitizer** — a Consumer that subscribes to `MessageSubmitted` events, strips PII into a `SanitizedMessage`, and emits `MessageSanitized` back to the entity.
- **InputGuardrail** — registered on the agent's `before-agent-invocation` hook; enforces the tool-policy allowlist before the agent's first LLM call on any task.
- **OutputGuardrail** — registered on the agent's `after-llm-response` hook; validates the structured `ModerationDecision` before it leaves the agent loop.
- **ModerationView + ModerationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded message set for your own (the JSONL file under `src/main/resources/sample-events/messages.jsonl` after generation).
- `SPEC.md §5` — extend `ModerationDecision` with deployment-specific fields (e.g., `regulatoryCategory`, `escalationTier`, `appealReference`).
- `prompts/moderation-agent.md` — narrow the agent's role to a specific content domain (financial-advice moderation, healthcare-claim moderation, etc.).
- `eval-matrix.yaml` — wire a real PII redactor by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a message + policy set → it is sanitized → moderated → the decision appears in the UI.
2. The agent attempts to invoke a tool that is not on the policy allowlist → the `before-agent-invocation` guardrail rejects the invocation → the agent retries without the forbidden tool → a decision lands.
3. The agent returns a malformed decision → the `after-llm-response` guardrail rejects it → the agent retries → a well-formed decision lands.
4. PII strings submitted in the message never appear in the LLM call log; only the redacted form does.

## License

Apache 2.0.
