# Akka Sample: Pull Request Review Agent

A reviewer agent analyzes a pull request diff and generates structured code-review feedback; a documentation-alignment agent checks whether the change is consistent with the project's documented contracts and conventions; the workflow loops until the reviewer accepts the feedback quality or the retry ceiling is reached. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the inbound PR stream and the outbound review surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.dev-code.pr-reviewer  ~/my-projects/pr-reviewer
cd ~/my-projects/pr-reviewer
```

(Optional) Edit `SPEC.md` to change the review rubric, the documentation sources the alignment agent checks, or the retry ceiling.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReviewerAgent** — AutonomousAgent that analyzes a PR diff and produces structured feedback comments, taking prior alignment notes when revising.
- **AlignmentAgent** — AutonomousAgent that checks the generated feedback against the project's documented contracts and style conventions; returns either `APPROVE` or `REVISE` with a typed `AlignmentNotes` payload.
- **ReviewWorkflow** — Workflow that runs the review → alignment-check → revise loop up to a configurable retry ceiling, transitions the review to `APPROVED` on alignment acceptance or to `REJECTED_FINAL` when the ceiling is hit.
- **ReviewEntity** — EventSourcedEntity that holds the review lifecycle, every feedback draft, every alignment check, and the final outcome.
- **PrQueue** — EventSourcedEntity that logs each PR submission for replay and audit.
- **ReviewsView** — read-side projection that the UI lists and streams via SSE.
- **PrSubmissionConsumer** — Consumer that starts a workflow per inbound PR submission.
- **PrSimulator** — TimedAction that drips a sample PR diff every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **ReviewEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the PR diffs the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Review` record fields (e.g., raise `maxAttempts`, change the feedback comment limit).
- `prompts/reviewer.md` — narrow the review criteria (e.g., focus only on security or performance).
- `prompts/alignment.md` — change the documentation sources the alignment agent checks.
- `eval-matrix.yaml` — tighten enforcement on the personal-critique guardrail (currently blocking).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a PR diff → review progresses `REVIEWING` → `CHECKING_ALIGNMENT` → `APPROVED` within the retry ceiling.
2. Force-fail alignment check → review hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every feedback draft and every alignment check for audit.
3. The personal-critique guardrail blocks feedback that contains personal language directed at the author, so the alignment agent never sees non-compliant feedback.
4. Each completed cycle emits an `EvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
