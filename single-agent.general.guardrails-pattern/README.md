# Akka Sample: Guardrails

A single-agent pattern showing how to attach input and output guardrails to an AI agent. The agent accepts free-form user prompts and produces structured responses. A `before-llm-call` guardrail screens the incoming prompt for policy violations before the model ever sees it; a `before-agent-response` guardrail validates the candidate reply's structure before it leaves the agent loop.

Demonstrates the **single-agent** coordination pattern with two guardrail hooks — the minimal building blocks any production agent should carry.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — all guardrail logic is in-process.

## Generate the system

```sh
cp -r ./single-agent.general.guardrails-pattern  ~/my-projects/guardrails-pattern
cd ~/my-projects/guardrails-pattern
```

(Optional) Edit `SPEC.md` to replace the default policy vocabulary with your own blocked-topic list, or to tighten the output schema the response guardrail validates.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AssistantAgent** — an AutonomousAgent that receives a user prompt and returns a typed `AgentReply`.
- **PromptGuardrail** — a `before-llm-call` hook that screens every incoming prompt against a configurable policy vocabulary before the model call is issued.
- **ReplyGuardrail** — a `before-agent-response` hook that validates every candidate reply for structural integrity and topic compliance before it leaves the agent.
- **InteractionWorkflow** — orchestrates prompt-guard → agent-call → reply-guard per submitted prompt.
- **InteractionEntity** — an EventSourcedEntity holding the per-interaction lifecycle.
- **InteractionView + InteractionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded blocked-topic vocabulary for your domain (e.g., financial-advice topics for a banking deployer, medical-diagnosis terms for a healthcare deployer).
- `SPEC.md §5` — extend `AgentReply` with domain-specific structured fields.
- `prompts/assistant-agent.md` — narrow the agent's persona and response format.
- `eval-matrix.yaml` — wire a real topic classifier or regex library by naming it in the `PromptGuardrail` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a clean prompt → the input guardrail passes → the agent replies → the reply guardrail passes → the structured reply appears in the UI.
2. A user submits a prompt that triggers a blocked topic → the `before-llm-call` guardrail blocks it → no model call is made → the UI shows a `BLOCKED` status.
3. The agent produces a malformed reply on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed reply lands; the UI never displays the malformed version.
4. Both guardrail events are visible in the interaction timeline on the detail pane.

## License

Apache 2.0.
