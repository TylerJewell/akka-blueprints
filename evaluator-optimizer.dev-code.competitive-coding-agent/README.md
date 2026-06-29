# Akka Sample: USACO Competitive Programming Agent

A solver agent reads a USACO-style problem statement; a judge agent runs the generated code against all test cases in a resource-bounded sandbox and returns either a `PASS` verdict with the accepted solution or a `FAIL` verdict with structured failure notes; the two iterate until the code passes all cases or the retry ceiling is hit. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **E2B** with `E2B_API_KEY` set — the sandbox step executes generated code inside a remote E2B micro-VM. Alternatively, replace the sandbox backend with **Modal** or **Robocorp** by editing `SPEC.md §4` before running `/akka:specify`.

## Generate the system

```sh
cp -r ./evaluator-optimizer.dev-code.competitive-coding-agent  ~/my-projects/competitive-coding-agent
cd ~/my-projects/competitive-coding-agent
```

(Optional) Edit `SPEC.md` to change the problem library, the per-submission time limit, the memory ceiling, or the maximum retry count.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SolverAgent** — AutonomousAgent that reads a problem statement (plus optional prior failure notes) and produces a complete Java solution.
- **JudgeAgent** — AutonomousAgent that analyzes a sandbox execution report and returns either `PASS` or `FAIL` with a typed `FailureNotes` payload.
- **SolvingWorkflow** — Workflow that runs the generate → sandbox-gate → judge → revise loop up to a configurable retry ceiling, transitions the submission to `ACCEPTED` on a PASS verdict or to `REJECTED_FINAL` when the ceiling is hit.
- **SubmissionEntity** — EventSourcedEntity that holds the submission lifecycle, every generated solution, every sandbox report, every judge verdict, and the final outcome.
- **ProblemQueue** — EventSourcedEntity that logs each problem submission for replay and audit.
- **SubmissionsView** — read-side projection that the UI lists and streams via SSE.
- **ProblemConsumer** — Consumer that starts a workflow per inbound submission.
- **ProblemSimulator** — TimedAction that drips a sample USACO problem every 90 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **SolverEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the problems the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Submission` record fields (e.g., raise `maxAttempts`, change the time-limit ceiling).
- `prompts/solver.md` — narrow the generation strategy (e.g., force a specific algorithm family).
- `prompts/judge.md` — change the verdict rubric (e.g., add partial-credit scoring).
- `eval-matrix.yaml` — tighten the sandbox guardrail (currently blocking) or adjust the CI-gate scope.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a problem → submission progresses `GENERATING` → `JUDGING` → `ACCEPTED` within the retry ceiling.
2. Force-fail test cases → submission hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every solution and every judge report for audit.
3. The sandbox guardrail blocks a solution that exceeds resource limits before the judge ever scores it.
4. Each completed cycle emits an `EvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
