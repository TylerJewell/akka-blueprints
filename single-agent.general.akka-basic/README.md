# Akka Sample: Single-Prompt Workflow

A single workflow wraps one LLM call — accepting a user prompt, invoking the model, running an after-response guardrail to validate the output format, and returning a typed result. The LLM call rides inside a durable `Workflow` step so the runtime retries on transient failure and records the outcome in an `EventSourcedEntity`.

Demonstrates the **single-agent** coordination pattern in its most direct form: one model call, one durable step, one governance check on the way out.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the agent and guardrail are self-contained.

## Generate the system

```sh
cp -r ./single-agent.general.akka-basic  ~/my-projects/akka-basic
cd ~/my-projects/akka-basic
```

(Optional) Edit `SPEC.md` to adjust the prompt schema or swap the response type for a more domain-specific record.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PromptAgent** — an AutonomousAgent that accepts a user prompt and context parameters, then returns a typed `PromptResult`.
- **PromptWorkflow** — one workflow per run; single step calls the agent, retries on transient failure, records the result.
- **PromptEntity** — an EventSourcedEntity holding the per-run lifecycle (SUBMITTED → PROCESSING → COMPLETED / FAILED).
- **OutputGuardrail** — an after-agent-response guardrail that checks the result's required fields are populated and the confidence score is in range before the response leaves the agent loop.
- **PromptView + PromptEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the prompt categories (currently: summarisation, extraction, classification) to match your use case.
- `SPEC.md §5` — extend `PromptResult` with domain-specific fields (e.g., `extractedEntities`, `labelScores`, `citedPassages`).
- `prompts/prompt-agent.md` — narrow the agent's behaviour to a specific task family; the default instructs the agent to handle all three seeded categories.
- `eval-matrix.yaml` — extend the `after-llm-response` guardrail's checks if your `PromptResult` fields require domain rules beyond structural validation.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a prompt → the workflow runs → a typed result appears in the UI within 30 s.
2. The agent returns a structurally invalid result on first try → the `after-agent-response` guardrail rejects it → the agent retries → a valid result lands.
3. A result whose confidence score is out of range (e.g. 1.5) is caught by the guardrail, not stored.
4. Concurrent runs for independent prompt IDs do not interfere — each workflow has its own entity.

## License

Apache 2.0.
