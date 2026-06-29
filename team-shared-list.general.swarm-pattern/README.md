# Akka Sample: Swarm Pattern

A coordinator agent splits a job into work items on a shared list; worker agents pick up items on their own, process them, and write results back — demonstrating the **team-shared-list** coordination pattern without any external orchestrator.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: **None**. The shared list, the worker pool, the result store, and the progress monitor are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./team-shared-list.general.swarm-pattern  ~/my-projects/swarm-pattern
cd ~/my-projects/swarm-pattern
```

(Optional) Edit `SPEC.md` to change the job kinds the simulator submits, the number of worker agents, or the model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **Coordinator** — AutonomousAgent that decomposes a job brief into a list of work items.
- **Worker** — AutonomousAgent (run as several instances) that claims one work item, produces a result, and signals when it cannot continue without another worker's output.
- **DecompositionWorkflow** — Workflow that runs the coordinator and writes one work item per spec onto the shared list.
- **WorkerWorkflow** — one durable loop per worker: poll the list, claim an open item atomically, produce a result, pass the quality gate, and mark the item done or stalled.
- **WorkItemEntity** — one EventSourcedEntity per work item; the atomic claim that prevents two workers grabbing the same item.
- **JobEntity**, **RelayMailbox**, **SubmissionLog** — job lifecycle, per-worker relay messages, and the submission audit log.
- **SwarmControl** — KeyValueEntity holding the operator pause flag.
- **WorkListView** — the shared read model the workers poll and the UI streams.
- **SwarmEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the job kinds the simulator drips, or remove the simulator entirely.
- `SPEC.md §11` — change the worker roster (`worker-1`, `worker-2`, `worker-3`) to a different count.
- `prompts/worker.md` — narrow the worker to a specific processing domain.
- `eval-matrix.yaml` — no controls are defined for this baseline; the file is a structural placeholder.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a job → the coordinator splits it into items → workers claim and complete them → the job reaches `FINISHED`.
2. Two workers claim concurrently → each item is claimed by exactly one worker; no double-claim.
3. A worker raises a relay request → the item goes `STALLED`, a message lands in the target worker's mailbox, the reply unblocks it.
4. The operator pauses the swarm → in-flight claiming pauses; resume continues the work.

## License

Apache 2.0.
