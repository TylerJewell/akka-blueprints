# Akka Sample: SDLC User Story Refiner

A `StoryDrafterAgent` generates a structured user story from a brief; a `ReviewerAgent` scores it against acceptance-criteria completeness, testability, and scope-tightness rules; the two exchange feedback until the reviewer accepts the story or the loop reaches its retry ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with an embedded quality eval control.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the inbound request stream and the outbound story surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.dev-code.story-refiner  ~/my-projects/story-refiner
cd ~/my-projects/story-refiner
```

(Optional) Edit `SPEC.md` to change the refinement rubric, the per-story field constraints, or the retry ceiling.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **StoryDrafterAgent** — AutonomousAgent that writes a structured user story (role, goal, benefit, acceptance criteria) from a brief, incorporating prior reviewer feedback on revision passes.
- **ReviewerAgent** — AutonomousAgent that scores a story draft against completeness, testability, and scope rules, returning either `APPROVE` or `REVISE` with a typed `ReviewNotes` payload.
- **RefinementWorkflow** — Workflow that runs the draft → review → revise loop up to a configurable retry ceiling, transitioning the story to `APPROVED` on reviewer acceptance or to `REJECTED_FINAL` when the ceiling is hit.
- **StoryEntity** — EventSourcedEntity holding the story lifecycle, every draft attempt, every review, and the final outcome.
- **RequestQueue** — EventSourcedEntity logging each submission for replay and audit.
- **StoriesView** — read-side projection that the UI lists and streams via SSE.
- **StoryRequestConsumer** — Consumer that starts a workflow per inbound submission.
- **RequestSimulator** — TimedAction that drips a sample brief every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **StoryEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the briefs the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Story` record fields (e.g., raise `maxAttempts`, change the required AC count).
- `prompts/story-drafter.md` — narrow the drafting style (e.g., BDD-format, Gherkin scenarios).
- `prompts/reviewer.md` — change the rubric (e.g., add a story-point-estimate dimension).
- `eval-matrix.yaml` — tighten enforcement on the quality eval (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a brief → story progresses `DRAFTING` → `REVIEWING` → `APPROVED` within the retry ceiling.
2. Force-fail rubric → story hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every attempt and every review for audit.
3. The quality eval records one `EvalRecorded` event per attempt, visible in the App UI's per-attempt timeline.
4. A story submitted with an over-constrained brief triggers the reviewer's scope feedback on attempt 1, and the drafter tightens scope on attempt 2.

## License

Apache 2.0.
