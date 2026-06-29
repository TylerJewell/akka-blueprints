# Akka Sample: Reflection Agent

A generator agent produces an initial response to a task; a reflection agent critiques that response and identifies specific improvements; the generator revises using the critique. The two alternate until the reflector accepts the output or the loop reaches its iteration ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The blueprint runs out of the box; the inbound task stream and the reflection surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.reflection-agent  ~/my-projects/reflection-agent
cd ~/my-projects/reflection-agent
```

(Optional) Edit `SPEC.md` to change the quality dimensions the reflector evaluates, the minimum acceptance score, or the iteration ceiling.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GeneratorAgent** — AutonomousAgent that produces an initial response to a task prompt; on revision calls, takes the reflector's structured critique and updates only the passages called out.
- **ReflectorAgent** — AutonomousAgent that scores a response against a fixed quality rubric and returns either `ACCEPT` with a rationale or `REVISE` with targeted change requests.
- **ReflectionWorkflow** — Workflow that runs the generate → reflect → revise loop up to a configurable iteration ceiling, transitions the task to `ACCEPTED` on reflector approval or to `REJECTED_FINAL` when the ceiling is hit.
- **TaskEntity** — EventSourcedEntity that holds the task lifecycle, every iteration's response and critique, and the final outcome.
- **SubmissionQueue** — EventSourcedEntity that logs each submitted task for replay and audit.
- **TasksView** — read-side projection that the UI lists and streams via SSE.
- **SubmissionConsumer** — Consumer that starts a workflow per inbound submission.
- **TaskSimulator** — TimedAction that drips a sample task every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-iteration eval event each cycle (control E1).
- **ReflectionEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the tasks the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Task` record fields (e.g., raise `maxIterations`, add a response-length ceiling).
- `prompts/generator.md` — narrow the domain the generator addresses (e.g., code explanations, structured summaries).
- `prompts/reflector.md` — change the rubric dimensions (e.g., add a citation-completeness check).
- `eval-matrix.yaml` — tighten enforcement on the eval-event (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task → response progresses `GENERATING` → `REFLECTING` → `ACCEPTED` within the iteration ceiling.
2. Force-fail rubric → task hits `REJECTED_FINAL` after the configured number of iterations; the entity preserves every iteration and every critique for audit.
3. Each completed iteration emits an `EvalRecorded` event that surfaces in the App UI's per-iteration timeline.
4. The reflector's structured critique is visible in the expanded view alongside the matching generated response.

## License

Apache 2.0.
