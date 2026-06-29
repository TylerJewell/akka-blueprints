# Akka Sample: Inline Agent

A single inline agent answers free-form questions at runtime — its system prompt, tools, and model are all defined in the request body, not baked into a class at compile time. The caller sends a question plus an agent definition; the agent returns a structured answer.

Demonstrates the **single-agent** coordination pattern using the inline-agent API: the agent definition is assembled at call time, allowing callers to vary instructions, tool lists, and output schemas per request without redeploying the service.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the agent definition and request examples live in-process.

## Generate the system

```sh
cp -r ./single-agent.general.inline-agent-demo  ~/my-projects/inline-agent-demo
cd ~/my-projects/inline-agent-demo
```

(Optional) Edit `SPEC.md` to change the seeded agent definitions (e.g., swap the Q&A agent for a classification agent or a summarisation agent).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **InlineAgentRunner** — a thin service component that receives an `InlineAgentRequest` (system prompt + user query + output schema), spins up an AutonomousAgent defined entirely from the request payload, runs it to completion, and returns a typed `AgentResponse`.
- **RunEntity** — an EventSourcedEntity holding per-run lifecycle: received → running → completed / failed.
- **RunWorkflow** — orchestrates the steps: validate → run agent → record result.
- **RunView + RunEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded agent definitions for your own (stored in `src/main/resources/sample-events/agent-definitions.jsonl`).
- `SPEC.md §5` — extend `AgentResponse` with domain-specific fields (e.g., `confidence`, `citations`, `followUpQueries`).
- `prompts/inline-agent-runner.md` — change the base instructions included with every inline request (a healthcare deployer might add HIPAA caveats; a financial deployer might add regulatory disclaimers).
- `eval-matrix.yaml` — this baseline has no controls; add your own once you know the deployment context.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A caller submits an inline agent definition with a question → the agent runs → a typed answer appears in the UI.
2. A caller submits a malformed definition (missing required field) → the service returns a 400 before spinning up any LLM call.
3. Two concurrent runs with different agent definitions execute independently without cross-contaminating each other's context.
4. The live list in the UI updates in real time as runs progress through states.

## License

Apache 2.0.
