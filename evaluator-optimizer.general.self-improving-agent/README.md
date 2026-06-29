# Akka Sample: Self-Improving Agent

An executor agent processes tasks and emits structured performance records; an optimizer agent analyzes those records and proposes prompt or tool-weight revisions; a workflow runs the reflect → propose → attest → apply loop until the modification passes a regression gate or the loop hits its ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; performance data is generated internally and the modification surface is modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.self-improving-agent  ~/my-projects/self-improving-agent
cd ~/my-projects/self-improving-agent
```

(Optional) Edit `SPEC.md` to change the task corpus the simulator drips, the regression threshold, or the retry ceiling.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ExecutorAgent** — AutonomousAgent that processes a task using the current prompt configuration and returns a typed `ExecutionResult` capturing output quality and latency signals.
- **OptimizerAgent** — AutonomousAgent that analyzes a batch of `PerformanceRecord` entries and proposes a `PromptRevision` — either a refined system-prompt fragment or an adjusted tool-call heuristic.
- **ImprovementWorkflow** — Workflow that runs the execute → evaluate → propose → attest → apply loop up to a configurable retry ceiling, transitions the agent configuration to `APPLIED` on a passing attestation or to `REVISION_REJECTED` when the ceiling is hit.
- **AgentConfigEntity** — EventSourcedEntity that holds the current prompt configuration, every revision proposal, every attestation verdict, and the active configuration lineage.
- **TaskQueueEntity** — EventSourcedEntity that logs each submitted task for replay and audit.
- **ConfigView** — read-side projection that the UI lists and streams via SSE.
- **TaskResultConsumer** — Consumer that starts an improvement workflow per completed task batch.
- **TaskSimulator** — TimedAction that drips sample tasks every 60 s so the App UI is never empty.
- **AttestationSampler** — TimedAction that records the per-cycle attestation event each cycle (control E1).
- **AgentEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the tasks the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `AgentConfig` record fields (e.g., raise `maxRevisions`, change the regression threshold).
- `prompts/executor.md` — narrow the task domain (e.g., JSON extraction, code summarization).
- `prompts/optimizer.md` — change the revision strategy (e.g., require a diff instead of a full replacement).
- `eval-matrix.yaml` — tighten enforcement on the attestation gate (currently build-gate).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task batch → performance is evaluated → a revision is proposed → attestation passes → new config applied; the UI shows every cycle's result.
2. Force-fail the regression gate → the workflow cycles through all revisions and lands in `REVISION_REJECTED` with the best-performing config preserved and a structured rejection reason.
3. The eval event fires once per completed cycle, surfacing verdict, score, and whether the gate passed in the App UI timeline.
4. Attestation is recorded as a signed event before any config change lands; no modification bypasses the gate.

## License

Apache 2.0.
