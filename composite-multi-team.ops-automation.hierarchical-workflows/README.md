# Akka Sample: Hierarchical Workflow Automation

An orchestrator agent breaks a cross-system operations request into structured tasks, delegates those tasks to specialist sub-teams — a discovery team that fans out across system inventories, an execution team whose workers claim tasks from a shared board, and a validation team whose validators each score the result — and assembles a final operations report after a guardrail vets the output. Demonstrates the **composite-multi-team** coordination pattern with embedded governance across tool-calling agents that reach across system boundaries.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. The discovery team, the execution board, the validation panel, and the request intake are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./composite-multi-team.ops-automation.hierarchical-workflows  ~/my-projects/hierarchical-workflow-automation
cd ~/my-projects/hierarchical-workflow-automation
```

(Optional) Edit `SPEC.md` to change the operations domains the simulator drips, the executor roster, the validation panel axes, or the model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **Orchestrator** — AutonomousAgent that decomposes an operations request into a structured `WorkflowPlan` and, at the end, assembles the final `OpsReport` under an output guardrail.
- **DiscoveryLead + SystemScanner** — the discovery team: the lead plans which systems to scan and what to look for; several scanner instances each produce one `ScanResult`. The lead then synthesises the results into a `DiscoverySummary`. This is the delegation capability.
- **ExecutionLead + TaskExecutor** — the execution team: the lead plans the execution tasks onto a shared board; a roster of executor loops each claim an open task, run the action via `SystemTools`, and mark it complete. Executors self-organising around a shared task list is the team capability.
- **Validator** — the validation team: several validator instances each score the execution result on one axis (correctness, safety, compliance); a deterministic aggregation rule turns their notes into a pass-or-retry verdict. A scored panel feeding a deterministic rule is the moderation capability.
- **WorkflowInstance** — the orchestrator pipeline: plan → discovery → execution → validation → report, with a retry loop when validation requests corrections.
- **WorkflowEntity + TaskEntity** — the shared workflow state and the per-task board entry whose atomic claim prevents two executors grabbing the same task.
- **WorkflowBoardView, TaskBoardView** — the request list and the per-task read model the UI streams.
- **RequestConsumer** — fires a WorkflowInstance per queued request.
- **AuditEvalConsumer** — fires a non-blocking quality eval on every stage result.
- **WorkflowEndpoint + AppEndpoint** — REST + SSE + static UI serving, including the post-execution compliance-review surface.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the operations domains the simulator drips, or remove the simulator entirely.
- `SPEC.md §11` — change the executor roster (`executor-1`, `executor-2`) or the validation panel axes (`correctness`, `safety`, `compliance`).
- `prompts/system-scanner.md`, `prompts/task-executor.md`, `prompts/validator.md` — narrow each team to a specific system domain.
- `eval-matrix.yaml` — extend the before-tool-call guardrail's refused-operation list or change the audit-eval thresholds.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an operations request → the orchestrator plans a workflow → the discovery team delegates and synthesises → the execution team fills tasks on a shared board → the validation panel scores the results → a passing verdict assembles and delivers the final report.
2. Two executors claim concurrently → each task is claimed by exactly one executor; no double-claim.
3. The validation panel returns a retry verdict → the failed tasks reset to open and re-execute, then the pipeline terminates.
4. An executor or scanner attempts a cross-system write it was not assigned → the before-tool-call guardrail blocks it before execution and the stage records the guardrail reason.
5. After a workflow reaches `COMPLETED`, an operator posts a post-execution compliance review → it is recorded against the report without changing the completed state.

## License

Apache 2.0.
