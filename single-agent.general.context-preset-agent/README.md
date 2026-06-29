# Akka Sample: Request Context Presets Agent

A single agent that reads a Request Context Preset — a combination of environment (`dev` / `staging` / `prod`) and caller role (`admin` / `guest`) — and dynamically adjusts its instructions, model selection, and tool availability before answering the caller's request. Admin callers in production get access to privileged actions; guest callers get a read-only surface; non-production environments can unlock diagnostic tools that production disables.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a `before-tool-call` guardrail that gates the `adminActionTool` exclusively to `admin`-preset callers, and a CI configuration gate that validates preset JSON schema correctness before any environment promotion.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — preset definitions live in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.general.context-preset-agent  ~/my-projects/context-preset-agent
cd ~/my-projects/context-preset-agent
```

(Optional) Edit `SPEC.md` to adjust the seeded presets — for example, add an `operator` role between `guest` and `admin`, or add a `canary` environment tier.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ContextPresetAgent** — an AutonomousAgent that resolves the caller's preset, selects its instructions and model from the preset definition, and gates tool calls based on the resolved role.
- **PresetRequestWorkflow** — orchestrates resolve-preset → execute-request → audit per inbound call.
- **PresetRequestEntity** — an EventSourcedEntity holding the per-request lifecycle.
- **PresetRegistry** — a Key-Value Entity that stores the named preset definitions (environment × role → instructions, model id, allowed tools).
- **PresetRequestView + PresetRequestEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded preset definitions in `src/main/resources/sample-events/presets.json`.
- `SPEC.md §5` — extend `PresetDefinition` with additional dimensions (e.g., `featureFlags`, `rateLimit`).
- `prompts/context-preset-agent.md` — narrow or widen the agent's role description for your deployment context.
- `eval-matrix.yaml` — add a real secrets resolver under the configuration-gate mechanism's implementation paragraph if you need to verify that presets don't embed credential values.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A `prod` + `admin` caller submits a request → the `adminActionTool` executes → the result appears in the UI.
2. A `prod` + `guest` caller requests the same action → the `before-tool-call` guardrail blocks `adminActionTool` → the agent returns a permission-denied explanation without the tool executing.
3. A preset JSON file with a missing required field fails the CI configuration gate before the preset reaches the agent runtime.
4. A `dev` + `admin` caller gets the `diagnosticTool` in their tool list; a `prod` + `admin` caller does not.

## License

Apache 2.0.
