# Akka Sample: ReWOO (Reasoning Without Observation)

A Planner writes the complete reasoning plan up front — including variable placeholders for tool results — before any tool runs. A Worker executes each tool call in sequence, filling in the variables. A Solver then reads the fully-populated plan and composes the final answer. Demonstrates the **planner-executor** coordination pattern with a fixed, observable plan and embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Tool calls — web lookups, file reads, code execution, calculation steps — are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.general.rewoo-fixed-plan  ~/my-projects/rewoo-fixed-plan
cd ~/my-projects/rewoo-fixed-plan
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that writes the full reasoning plan as a sequence of `PlanStep` records, each with a tool name, input expression (which may reference `#E<n>` variables from prior steps), and a result placeholder.
- **WorkerAgent** — AutonomousAgent that executes one plan step at a time, filling in variable references before calling the tool simulation.
- **SolverAgent** — AutonomousAgent that reads the completed plan (all `#E<n>` placeholders resolved) and produces the final answer.
- **QueryWorkflow** — Workflow with a plan → execute-steps-in-order → solve loop, plus a guardrail check before each tool call and terminal states for completion and failure.
- **QueryEntity** — EventSourcedEntity holding the query lifecycle, the full plan, and the solved answer.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **QueryView** — projection used by the UI.
- **QueryEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample queries the simulator submits, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Query` record fields (e.g., add a `confidenceScore` to `SolvedAnswer`).
- `prompts/planner.md` — narrow the planner to a specific knowledge domain.
- `eval-matrix.yaml` — add an `eval-event` control if you want per-step quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a query → planner writes the full plan, worker fills all variables, solver answers within ~3 minutes.
2. A plan step that would invoke a forbidden tool is rejected by the guardrail before execution; the query either completes via an alternate step or fails with a clear reason.
3. Trigger the operator halt → the in-flight tool call finishes; no further steps execute; query moves to `HALTED`.
4. A tool result containing a secret-shaped string is scrubbed before it is written into the plan's variable slot and before the Solver sees it.

## License

Apache 2.0.
