# Akka Sample: Shared-Memory Multi-Agent

Multiple agents share specific memory blocks — such as a project context block — so that a write by one agent is immediately visible to all others. An optional consolidation pass runs on a sleeptime schedule to merge concurrent writes before the next round of reads. Demonstrates the **team-shared-list** coordination pattern with embedded governance controls on shared-memory writes.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. The memory store, consolidation scheduler, and agent roster are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./team-shared-list.general.akka-shared-memory-team  ~/my-projects/shared-memory-multi-agent
cd ~/my-projects/shared-memory-multi-agent
```

(Optional) Edit `SPEC.md` to change the number of writer agents, the memory block names, or the consolidation schedule.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WriterAgent** — AutonomousAgent (run as several instances) that reads a memory block, contributes a knowledge fragment, and writes back an updated block; subject to a before-tool-call guardrail on each write.
- **ConsolidatorAgent** — AutonomousAgent that merges all pending fragments on a named block into a single coherent snapshot.
- **WriteWorkflow** — durable per-agent loop: read the shared block, call WriterAgent, pass the guardrail, write the fragment, schedule the next cycle.
- **ConsolidationWorkflow** — Workflow triggered by the consolidation schedule that runs ConsolidatorAgent and commits the merged block.
- **MemoryBlockEntity** — EventSourcedEntity; one instance per named memory block. Single-writer prevents concurrent fragment corruption.
- **FragmentEntity** — EventSourcedEntity; one per submitted fragment, holding content and provenance.
- **AgentRegistry** — KeyValueEntity listing the active writer roster.
- **MemoryBlockView** — the shared read model agents and the UI query.
- **FragmentConsumer** — Consumer that starts a consolidation pass when the fragment count on a block crosses a threshold.
- **ConsolidationScheduler** — TimedAction; triggers a sleeptime consolidation pass on every block that has unmerged fragments.
- **FragmentAgeMonitor** — TimedAction; releases stale unmerged fragments back to eligible status so consolidation does not stall.
- **MemoryEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the memory block names the simulator seeds, or remove the simulator entirely.
- `SPEC.md §11` — change the writer-agent roster (`writer-1`, `writer-2`, `writer-3`) to a different count.
- `prompts/writer-agent.md` — narrow the writer to a specific knowledge domain.
- `eval-matrix.yaml` — extend the before-tool-call guardrail's blocked-path list if you wire a real external memory store.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a memory block; writer agents post fragments; the block reaches `CONSOLIDATED`.
2. Two writers attempt to write the same block concurrently; each fragment is recorded exactly once (no duplicate write).
3. A writer attempts to overwrite a protected block section; the before-tool-call guardrail blocks the write; the fragment is recorded `REJECTED`.
4. A special-category attribute surfaces in a cross-agent fragment; the sanitizer removes it before the fragment is committed.
5. The consolidation pass merges all pending fragments into a single coherent block snapshot; older raw fragments are no longer visible in the active read model.

## License

Apache 2.0.
