# Akka Sample: SWE-Bench Coding Agent

A coding agent receives a GitHub issue and a repository snapshot; a patch agent produces a candidate fix; a test-evaluator agent runs the test suite and scores the patch against the ground truth; the two iterate until the tests pass or the loop hits its budget ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the issue intake and the test harness are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.dev-code.swebench-evaluator-loop  ~/my-projects/swebench-coding-agent
cd ~/my-projects/swebench-coding-agent
```

(Optional) Edit `SPEC.md` to change the patch strategy instructions, the test-pass threshold, the token budget, or the iteration ceiling.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PatchAgent** — AutonomousAgent that reads a GitHub issue plus the relevant file context and produces a candidate unified diff (patch).
- **TestEvaluatorAgent** — AutonomousAgent that scores a patch by running the test suite in a sandboxed environment and returning a `TestResult` with pass/fail counts and a `PASS` or `FAIL` verdict.
- **SolvingWorkflow** — Workflow that runs the patch → ci-gate → evaluate → revise loop up to a configurable iteration ceiling, transitions the issue to `SOLVED` on a passing test run or to `EXHAUSTED` when the ceiling is hit.
- **IssueEntity** — EventSourcedEntity that holds the issue lifecycle, every patch attempt, every test result, and the final outcome.
- **IssueQueue** — EventSourcedEntity that logs each submitted issue for replay and audit.
- **IssuesView** — read-side projection that the UI lists and streams via SSE.
- **IssueIngestConsumer** — Consumer that starts a workflow per inbound submission.
- **IssueSimulator** — TimedAction that drips a sample issue every 90 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-iteration eval event each cycle (control E1).
- **IssueEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the issues the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Issue` record fields (e.g., raise `maxIterations`, change the token budget ceiling).
- `prompts/patch-agent.md` — narrow the patch strategy (e.g., restrict to single-file edits, add language-specific constraints).
- `prompts/test-evaluator-agent.md` — change the scoring rubric (e.g., require 100% pass rather than threshold).
- `eval-matrix.yaml` — tighten enforcement on the ci-gate (currently blocking).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an issue → issue progresses `PATCHING` → `EVALUATING` → `SOLVED` within the iteration ceiling.
2. Force-fail the test suite → issue hits `EXHAUSTED` after the configured number of iterations; the entity preserves every patch and every test result for audit.
3. The CI gate blocks a malformed patch before the test evaluator runs, so the evaluator never processes a structurally invalid diff.
4. Each completed cycle emits an `EvalRecorded` event that surfaces in the App UI's per-iteration timeline.

## License

Apache 2.0.
