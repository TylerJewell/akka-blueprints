# Akka Sample: Parallelization Workflow (Sectioning + Voting)

A supervisor dispatches identical or partitioned tasks to multiple worker agents running **in parallel**, then aggregates their results — either by merging independent sections into one output or by applying a voting rule to select or synthesise across diverse responses. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the input stream and worker tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.general.parallelization-vote-sectioning  ~/my-projects/parallelization
cd ~/my-projects/parallelization
```

(Optional) Edit `SPEC.md` to change the parallelization mode (sectioning vs. voting), worker count, or aggregation policy.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ParallelSupervisor** — AutonomousAgent that partitions a job into sections (or prepares a shared prompt for voting), then aggregates worker outputs.
- **SectionWorker** — AutonomousAgent that processes one section of the partitioned task and returns a typed section result.
- **VoteWorker** — AutonomousAgent that independently evaluates the full task and returns a typed vote result (used in voting mode).
- **ParallelWorkflow** — Workflow that fans work out to N workers concurrently, waits for all results (or a quorum), and calls the Supervisor for aggregation.
- **JobEntity** — EventSourcedEntity holding the full job lifecycle.
- **JobView** — projection the UI streams via SSE.
- **JobEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — switch between sectioning mode and voting mode, or run both in the same pipeline.
- `SPEC.md §5` — adjust the `Job` record fields (e.g., add `confidenceThreshold`).
- `prompts/supervisor.md` — tune the aggregation policy (majority vote, weighted score, section stitching).
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if workers call external tools.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a job → status enters `PARTITIONING`, then `IN_PROGRESS`, then `AGGREGATED`.
2. One worker times out → job enters `PARTIAL` with available section outputs; guardrail decides whether to surface or block.
3. Voting mode: all workers return; the supervisor applies the vote rule and the winning answer reaches the App UI.
4. Output guardrail blocks an aggregated result that contains a policy violation.

## License

Apache 2.0.
