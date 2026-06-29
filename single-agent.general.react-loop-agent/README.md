# Akka Sample: ReAct Agent Workflow

A single ReAct-pattern agent receives a user query and a set of available tools, then interleaves Thought/Action/Observation steps until it reaches a final answer or exhausts its iteration budget. Each tool call is governed before it executes; each reasoning chain is scored for hallucinated or circular steps after the answer lands.

Demonstrates the **single-agent** coordination pattern in the general domain. One `ReActAgent` (AutonomousAgent) drives the full loop; the surrounding components prepare its tool context, record every step, and audit the finished chain.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the tool registry lives in-process and the agent's tool calls are handled by local stubs.

## Generate the system

```sh
cp -r ./single-agent.general.react-loop-agent  ~/my-projects/react-loop-agent
cd ~/my-projects/react-loop-agent
```

(Optional) Edit `SPEC.md` to swap in a different seeded tool set — e.g., replace the three general-purpose stubs with domain-specific tools for a customer-support use case.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReActAgent** — an AutonomousAgent that receives a query and iterates Thought/Action/Observation steps, calling registered tools as actions and recording each observation before deciding whether to continue or return a final answer.
- **ReActWorkflow** — orchestrates the full run lifecycle: initialize → loop → finalize.
- **RunEntity** — an EventSourcedEntity holding the per-run state and full step trace.
- **ToolDispatcher** — a Consumer that subscribes to `ActionRequested` events, executes the named tool, and emits `ObservationRecorded` back to the entity.
- **RunView + RunEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded tool stubs for real integrations (web search, code interpreter, calculator, database lookup).
- `SPEC.md §5` — extend `Step` with additional fields (e.g., `toolVersion`, `latencyMs`, `confidenceScore`).
- `prompts/react-agent.md` — narrow the agent's persona and tool-use policy (e.g., cap observation length, forbid certain tool names in certain contexts).
- `eval-matrix.yaml` — wire the before-tool-call guardrail to a live policy store that checks tool permissions per authenticated user.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a query → the agent iterates Thought/Action/Observation → a final answer appears in the UI with the full step trace.
2. The agent requests a forbidden tool → the `before-tool-call` guardrail blocks it → the agent receives an error observation → it selects a permitted tool on the next iteration.
3. The agent produces a circular reasoning chain (repeats the same Thought twice without calling a tool) → the on-decision eval scores the run low and flags the chain.
4. A run that exhausts its iteration budget without reaching a final answer lands in `EXHAUSTED` state; the UI shows the partial trace.

## License

Apache 2.0.
