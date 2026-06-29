# Akka Sample: Durable Workflows with DBOS

A single workflow-recovery agent monitors a long-running durable operation, detects stalls or crashes mid-execution, and emits a structured recovery decision: RESUME / ABORT / ESCALATE plus a per-checkpoint status. The workflow state rides into the agent as a task attachment, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a graceful-degradation halt that resumes interrupted workflows rather than restarting them from scratch, and a periodic health evaluator that scores running workflow instances for latency drift and checkpoint progress.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — workflow state lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.ops-automation.durable-workflow-recovery  ~/my-projects/durable-workflow-recovery
cd ~/my-projects/durable-workflow-recovery
```

(Optional) Edit `SPEC.md` to point at a different set of seeded workflow definitions (e.g., switch from the generic ETL-pipeline checklist to a data-migration checklist or a payment-batch checklist).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WorkflowRecoveryAgent** — an AutonomousAgent that accepts workflow state as a task attachment plus checkpoint history and returns a typed `RecoveryDecision`.
- **RecoveryWorkflow** — orchestrates detect-stall → analyze → health-eval per monitored execution.
- **ExecutionEntity** — an EventSourcedEntity holding the per-execution lifecycle.
- **CheckpointConsumer** — a Consumer that subscribes to `CheckpointRecorded` events, checks for stall conditions, and emits `StallDetected` back to the entity.
- **ExecutionView + RecoveryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded workflow definitions for your own (the JSONL file under `src/main/resources/sample-events/workflow-definitions.jsonl` after generation).
- `SPEC.md §5` — extend `RecoveryDecision` with deployment-specific fields (e.g., `retryBudget`, `owningTeam`, `alertChannel`).
- `prompts/workflow-recovery-agent.md` — narrow the agent's role (a data-pipeline operator would constrain it to ETL-stage analysis; a payment processor to batch-settlement workflows).
- `eval-matrix.yaml` — configure health thresholds (stall timeout, latency drift ceiling) under the periodic-eval mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user registers a workflow execution → checkpoints arrive → a stall is detected → the agent issues a RESUME decision that appears in the UI.
2. An execution that fails repeatedly within its retry budget receives an ABORT decision, and the UI card transitions to ABORTED without manual intervention.
3. Every recorded decision has a periodic health score visible on the same UI card.
4. A crash mid-checkpoint is recovered from the last durable checkpoint, not from the start.

## License

Apache 2.0.
