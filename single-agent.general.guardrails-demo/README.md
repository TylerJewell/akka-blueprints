# Akka Sample: Agent with Guardrails Integration

A single conversational agent accepts free-text user messages and replies through two guardrail layers: an input topic-policy guardrail that blocks disallowed topics before the LLM call, and an output content-policy guardrail that validates the response before it leaves the agent loop. Both guardrails are wired directly into the agent — not as external post-processing steps.

Demonstrates the **single-agent** coordination pattern with two independent guardrail hooks: one `before-llm-call` and one `before-agent-response`. Together they show that a single agent can enforce both input and output policies without a second model.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the session store is in-process.

## Generate the system

```sh
cp -r ./single-agent.general.guardrails-demo  ~/my-projects/guardrails-demo
cd ~/my-projects/guardrails-demo
```

(Optional) Edit `SPEC.md` to change the blocked topic list or tighten the output-content policy rules.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ConversationAgent** — an AutonomousAgent with two guardrails registered: an input topic-policy guardrail (`TopicPolicyGuardrail`) and an output content-policy guardrail (`ContentPolicyGuardrail`).
- **SessionWorkflow** — orchestrates session-open → exchange → session-close; each exchange step calls the agent and records the result.
- **SessionEntity** — an EventSourcedEntity holding the per-session lifecycle and turn history.
- **SessionView + SessionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded blocked-topic list (default: financial-advice, medical-diagnosis, legal-advice).
- `SPEC.md §5` — extend `TurnRecord` with domain-specific fields (e.g., `sessionPurpose`, `userTier`, `languageCode`).
- `prompts/conversation-agent.md` — narrow the agent's persona (a product-support deployer would constrain it to questions about their product; a HR deployer would restrict it to policy lookups).
- `eval-matrix.yaml` — add regulation anchors once a jurisdiction is in scope.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user opens a session, sends a message on an allowed topic, and receives a well-formed reply — the session history shows the turn.
2. A user sends a message on a blocked topic — the `before-llm-call` guardrail intercepts it; the LLM is never called; the session records a `TurnBlocked` event.
3. The agent produces a response that violates the output content policy — the `before-agent-response` guardrail rejects it; the agent retries; the final reply satisfies the policy.
4. Service logs confirm no blocked-topic message body ever appeared in an LLM call log.

## License

Apache 2.0.
