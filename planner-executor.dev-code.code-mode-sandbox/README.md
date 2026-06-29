# Akka Sample: Sandboxed Code-Execution Capability

A Planner decomposes a coding task into a sequence of sandbox invocations, each governed by a capability token. An Executor writes the code for each step. Before any sandbox call, the harness checks the token; before any prompt reaches the sandbox compiler, secrets are stripped. Demonstrates the **planner-executor** coordination pattern with capability-based authorization and prompt sanitization.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (browser-or-sandbox-execution): **None.** The sandbox is a deterministic in-process interpreter that runs code in a restricted JVM sandbox; no Docker or external runtime is required for the sample.

## Generate the system

```sh
cp -r ./planner-executor.dev-code.code-mode-sandbox  ~/my-projects/code-mode-sandbox
cd ~/my-projects/code-mode-sandbox
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or capability token scopes.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that decomposes a coding task into a `ExecutionPlan` (ordered list of `SandboxStep` items, each carrying required capability scopes).
- **ExecutorAgent** — AutonomousAgent that writes source code for a single `SandboxStep` and returns a `CodePayload`.
- **ExecutionWorkflow** — Workflow with a plan → sanitize-prompt → gate → execute → record → decide loop, replan branch, and terminal exit states.
- **ExecutionJobEntity** — EventSourcedEntity holding the job lifecycle, execution plan, step results, and final output.
- **CapabilityRegistryEntity** — EventSourcedEntity holding the active capability token set for a deployment; validated by the guardrail step.
- **JobQueue** — EventSourcedEntity that is the audit log of submitted jobs.
- **ExecutionJobView** — projection used by the UI.
- **JobRequestConsumer** — Consumer that starts a workflow per submission.
- **JobSimulator** — TimedAction that drips sample jobs every 90 s.
- **StaleJobMonitor** — TimedAction that marks jobs stuck in EXECUTING for more than 5 minutes.
- **JobEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample job prompts or remove the simulator.
- `SPEC.md §5` — adjust the `ExecutionJob` record fields (e.g., add `languageTarget`).
- `prompts/planner.md` — narrow the planner to a specific language ecosystem.
- `eval-matrix.yaml` — add an `eval-event` control for per-step output quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a coding task → planner decomposes into steps, executor writes each, sandbox runs each, job completes within ~3 minutes.
2. Submit a task whose plan requires a capability scope the registry does not grant → the guardrail blocks the step; the planner replans within the granted scopes or the job fails gracefully.
3. Submit a task whose prompt contains a credential-shaped string → the sanitizer strips it before the prompt reaches the executor; the stripped form is what appears in the execution plan and in subsequent prompts.
4. Submit a task and revoke the capability token mid-execution → the next step is blocked; the job ends in BLOCKED with a clear reason.

## License

Apache 2.0.
