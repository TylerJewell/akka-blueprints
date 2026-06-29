# Akka Sample: Sleeptime Memory-Consolidation Agent

A background consolidation agent shares memory blocks with a primary agent and, every N interaction steps, asynchronously rewrites those shared blocks to distil and compact learned context. Demonstrates the **continuous-monitor** coordination pattern wired with three governance mechanisms (before-tool-call guardrail, HOTL runtime monitoring, periodic eval for drift detection).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./continuous-monitor.general.akka-sleeptime-consolidation  ~/my-projects/sleeptime-consolidation
cd ~/my-projects/sleeptime-consolidation
```

(Optional) Edit `SPEC.md` to change the consolidation trigger interval (`N` steps), the memory block schema, or to attach a real persistent store for memory blocks.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PrimaryAgent** — Agent that handles user requests and reads from shared `MemoryBlock` entities.
- **SleeptimeConsolidatorAgent** — AutonomousAgent that rewrites memory blocks to consolidate learned context.
- **ConsolidationWorkflow** — per-block Workflow orchestrating the consolidation cycle with a before-tool-call guardrail.
- **MemoryBlockEntity** — EventSourcedEntity holding each block's full version history (created → updated → consolidating → consolidated → stale).
- **InteractionCounterEntity** — EventSourcedEntity tracking per-session step counts; triggers consolidation every N steps.
- **ConsolidationTrigger** — TimedAction that periodically checks for blocks needing consolidation and starts `ConsolidationWorkflow` instances.
- **DriftEvalRunner** — TimedAction that samples recently consolidated blocks and scores semantic drift vs. the prior version.
- **MemoryView + MemoryEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the consolidation interval (default: every 5 steps) or replace the in-memory interaction counter with a real session store.
- `SPEC.md §5` — extend `MemoryBlock` with domain-specific fields (`topicTags`, `confidenceScore`, etc.).
- `prompts/sleeptime-consolidator.md` — tune the rewriting instructions for your memory schema.
- `eval-matrix.yaml` — add a `regulation_anchor` if your deployment falls under an AI governance framework.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. After N interaction steps, `ConsolidationTrigger` fires and a `ConsolidationWorkflow` begins for the active memory block.
2. `SleeptimeConsolidatorAgent` rewrites the block; the new version is stored and the block status transitions to CONSOLIDATED.
3. A memory-mutating tool call attempted outside an active consolidation is blocked by the before-tool-call guardrail.
4. `DriftEvalRunner` scores a consolidated block for semantic drift; the score appears in the UI.

## License

Apache 2.0.
