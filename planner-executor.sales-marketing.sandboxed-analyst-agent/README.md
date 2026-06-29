# Akka Sample: Sales Call Data Analyzer (Sandbox)

An AnalystAgent plans an analysis of uploaded sales call data, executes each analysis step inside a sandboxed code-execution environment, records results on an analysis ledger plus a progress ledger, and replans when a step fails. Demonstrates the **planner-executor** coordination pattern with sandboxed code execution and embedded governance, including a PII sanitizer, a before-execution guardrail, and an automatic safety halt.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration tier: **E2B** for sandboxed code execution. Requires an `E2B_API_KEY` environment variable pointing to an active E2B account. Alternatively choose the mock-execution path during scaffolding — no external account needed.

## Generate the system

```sh
cp -r ./planner-executor.sales-marketing.sandboxed-analyst-agent  ~/my-projects/sandboxed-analyst-agent
cd ~/my-projects/sandboxed-analyst-agent
```

(Optional) Edit `SPEC.md` to change the analysis workflow, model provider, or sandbox behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AnalystAgent** — AutonomousAgent that maintains an analysis ledger (dataset facts, hypotheses, plan, current execution step) and a progress ledger (per-step attempt counts, verdicts, code outputs). Decides which analysis step to run next. Replans on three consecutive failures.
- **SandboxExecutorAgent** — AutonomousAgent that submits Python analysis scripts to the E2B sandbox (or mock executor) and returns typed `ExecutionResult` records.
- **AnalysisWorkflow** — Workflow with a plan → dispatch → guardrail → execute → sanitize → record → decide loop, replan branch, and terminal exit states.
- **AnalysisJobEntity** — EventSourcedEntity holding the job lifecycle, both ledgers, and the final report.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **UploadQueue** — EventSourcedEntity acting as the audit log of uploaded datasets.
- **AnalysisJobView** — projection used by the UI.
- **UploadConsumer** — Consumer that subscribes to UploadQueue events and starts a workflow per upload.
- **DatasetSimulator** — TimedAction that drips a sample dataset upload every 90 seconds.
- **StaleJobMonitor** — TimedAction that marks jobs stuck in EXECUTING past 5 minutes as STUCK.
- **AnalysisEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the dataset prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `AnalysisJob` record fields (e.g., add `confidenceScore`).
- `prompts/analyst.md` — narrow the analyst to a specific analysis type (e.g., churn-only).
- `eval-matrix.yaml` — add an `eval-event` control if you want per-step quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Upload a sales call CSV → analyst plans, executes analysis steps, completes within ~3 minutes. UI reflects each transition via SSE. The expanded view shows an analysis ledger with a non-empty plan, a progress ledger with 3–8 entries, and a non-empty `AnalysisReport`.
2. Inject a code-policy violation → guardrail blocks the offending execution step; analyst records the block on the progress ledger and replans.
3. Trigger the operator halt → no new execution steps dispatch; in-flight steps finish; job moves to `HALTED`.
4. A step result containing a customer phone number is scrubbed by the PII sanitizer before it reaches the analyst's next-step prompt.

## License

Apache 2.0.
