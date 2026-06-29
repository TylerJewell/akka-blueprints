# Akka Sample: SDLC Technical Designer

A single technical-design agent takes a feature description and a project context profile, then produces a structured architecture design: chosen components, data model, API surface, and a decision log explaining each choice. The feature description enters as a task attachment; the output is a typed `DesignProposal` suitable for review and downstream code generation.

Demonstrates the **single-agent** coordination pattern in the dev-code domain. One `DesignAgent` (AutonomousAgent) carries the full design decision; the surrounding components prepare its input, enforce output structure, and evaluate decision quality. Two governance mechanisms are wired around the agent: a `before-agent-response` guardrail that validates the proposal's schema and internal consistency, and an on-decision evaluator that scores every proposal for completeness and rationale depth.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No integration-tier host software required. The blueprint runs out of the box — the project context corpus lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.dev-code.sdlc-designer  ~/my-projects/sdlc-designer
cd ~/my-projects/sdlc-designer
```

(Optional) Edit `SPEC.md` to point at a different project-context profile (e.g., swap the seeded microservices context for a mobile app context or a data-pipeline context).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DesignAgent** — an AutonomousAgent that accepts a feature description (as a task attachment) plus a project context profile and returns a typed `DesignProposal`.
- **DesignWorkflow** — orchestrates context-loading → design → eval per submitted design request.
- **DesignRequestEntity** — an EventSourcedEntity holding the per-request lifecycle.
- **ContextLoader** — a Consumer that subscribes to `DesignRequestSubmitted` events, assembles the project context package, and emits `ContextLoaded` back to the entity.
- **DesignView + DesignEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded project context profiles for your own (the JSONL file under `src/main/resources/sample-events/project-contexts.jsonl` after generation).
- `SPEC.md §5` — extend `DesignProposal` with team-specific fields (e.g., `targetLanguage`, `cloudProvider`, `complianceTier`).
- `prompts/design-agent.md` — narrow the agent's role (a platform team would constrain it to internal service mesh patterns; a startup team would bias toward minimal-footprint designs).
- `eval-matrix.yaml` — wire a real schema validator by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a feature description + project context → context is loaded → the design proposal appears in the UI.
2. The agent returns a malformed proposal on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed proposal lands.
3. Every recorded proposal has an on-decision eval score visible on the same UI card.
4. A proposal whose decision-log entries are empty receives an eval score of 1 and a flagged card.

## License

Apache 2.0.
