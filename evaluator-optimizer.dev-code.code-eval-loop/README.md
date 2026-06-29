# Akka Sample: Code Assistant (AlphaCodium-style)

A generator agent produces code from a problem statement; a verifier agent runs import checks and test execution against it; the two iterate until all tests pass or the retry budget is exhausted. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; code execution is sandboxed inside the service and no external runtime is required.

## Generate the system

```sh
cp -r ./evaluator-optimizer.dev-code.code-eval-loop  ~/my-projects/code-eval-loop
cd ~/my-projects/code-eval-loop
```

(Optional) Edit `SPEC.md` to change the test runner, the retry budget, or the guardrail syscall deny-list.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GeneratorAgent** — AutonomousAgent that produces Java code for a problem statement, accepting prior verifier feedback when revising.
- **VerifierAgent** — AutonomousAgent that runs import checks and a sandboxed test execution, returns `PASS` or `FAIL` with a typed `VerificationNotes` payload.
- **SolutionWorkflow** — Workflow that runs the generate → sandbox-check → verify loop up to a configurable retry ceiling, transitions the solution to `PASSED` on a full test pass or to `EXHAUSTED` when the budget runs out.
- **SolutionEntity** — EventSourcedEntity that holds the solution lifecycle, every attempt's code, every verification result, and the final outcome.
- **ProblemQueue** — EventSourcedEntity that logs each submission for replay and audit.
- **SolutionsView** — read-side projection that the UI lists and streams via SSE.
- **ProblemConsumer** — Consumer that starts a workflow per inbound submission.
- **ProblemSimulator** — TimedAction that drips a sample problem every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **CodeEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the problems the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Solution` record fields (e.g., raise `maxAttempts`, change the default language).
- `prompts/generator.md` — narrow the language target (e.g., Python only) or add style constraints.
- `prompts/verifier.md` — change the verification rubric (e.g., require coverage thresholds).
- `eval-matrix.yaml` — tighten the sandbox guardrail from blocking to hard-fail on first syscall violation.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a problem statement → solution progresses `GENERATING` → `VERIFYING` → `PASSED` within the retry budget.
2. Force-fail tests → solution hits `EXHAUSTED` after the configured number of attempts; the entity preserves every attempt and every verification result for audit.
3. The sandbox guardrail blocks code that attempts a forbidden syscall or unsafe import; the generator re-drafts without the violation.
4. Each completed cycle emits an `EvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
