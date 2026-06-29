# Akka Sample: Evaluator-Optimizer Loop

A generator agent produces a candidate solution to a problem statement; an evaluator agent scores it against a quality rubric and returns feedback; the two iterate until the evaluator accepts the result or the loop reaches its attempt ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the inbound problem stream and the result surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.evaluator-optimizer-loop  ~/my-projects/evaluator-optimizer-loop
cd ~/my-projects/evaluator-optimizer-loop
```

(Optional) Edit `SPEC.md` to change the problem topics the simulator drips, the acceptance rubric, the quality score threshold, or the attempt ceiling.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GeneratorAgent** — AutonomousAgent that produces a candidate solution to a problem statement, accepting prior evaluator feedback when revising.
- **EvaluatorAgent** — AutonomousAgent that scores a candidate against the quality rubric, returns either `ACCEPT` or `REVISE` with a typed `EvaluationNotes` payload.
- **OptimizationWorkflow** — Workflow that runs the generate → evaluate → revise loop up to a configurable attempt ceiling, transitions the job to `ACCEPTED` on evaluator approval or to `REJECTED_FINAL` when the ceiling is hit.
- **JobEntity** — EventSourcedEntity that holds the job lifecycle, every attempt's candidate, every evaluation, and the final outcome.
- **SubmissionQueue** — EventSourcedEntity that logs each submission for replay and audit.
- **JobsView** — read-side projection that the UI lists and streams via SSE.
- **SubmissionConsumer** — Consumer that starts a workflow per inbound submission.
- **ProblemSimulator** — TimedAction that drips a sample problem every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **OptimizationEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the problem statements the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Job` record fields (e.g., raise `maxAttempts`, change the quality score threshold).
- `prompts/generator.md` — narrow the domain the generator addresses (e.g., code generation, structured summaries, formal proofs).
- `prompts/evaluator.md` — change the rubric dimensions.
- `eval-matrix.yaml` — tighten enforcement on the output guardrail (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a problem → job progresses `GENERATING` → `EVALUATING` → `ACCEPTED` within the attempt ceiling.
2. Force-fail rubric → job hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every attempt and every evaluation for audit.
3. The output guardrail blocks a candidate whose token count exceeds the ceiling, so the evaluator never sees over-length material.
4. Each completed cycle emits an `EvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
