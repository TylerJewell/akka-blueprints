# Akka Sample: Image Policy Scorer

A generator agent produces images from text prompts; a policy scorer agent evaluates each image against a configurable compliance rubric (brand safety, content appropriateness, platform policy); the two iterate until the scorer accepts the image or the loop reaches its attempt ceiling.
Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the image generation surface and the inbound prompt queue are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.content-editorial.image-policy-scorer  ~/my-projects/image-policy-scorer
cd ~/my-projects/image-policy-scorer
```

(Optional) Edit `SPEC.md` to change the policy rubric, the per-image size policy, the attempt ceiling, or the simulator prompt list.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GeneratorAgent** — AutonomousAgent that produces an image description and associated metadata from a text prompt; on revision calls it incorporates prior policy feedback.
- **ScorerAgent** — AutonomousAgent that evaluates an image description against the policy rubric and returns either `PASS` or `FAIL` with a typed `PolicyNotes` payload.
- **ScoringWorkflow** — Workflow that runs the generate → safety-gate → score → revise loop up to a configurable attempt ceiling, transitioning the image to `APPROVED` on scorer acceptance or to `REJECTED_FINAL` when the ceiling is hit.
- **ImageEntity** — EventSourcedEntity that holds the image lifecycle, every generation attempt, every policy verdict, and the final outcome.
- **PromptQueue** — EventSourcedEntity that logs each submitted prompt for replay and audit.
- **ImagesView** — read-side projection the UI lists and streams via SSE.
- **PromptConsumer** — Consumer that starts a workflow per inbound submission.
- **PromptSimulator** — TimedAction that drips a sample prompt every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **ImageEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Image` record fields (e.g., raise `maxAttempts`, change the default dimensions).
- `prompts/generator.md` — narrow the generation style or content category.
- `prompts/scorer.md` — change the rubric (e.g., add a platform-specific policy rule).
- `eval-matrix.yaml` — tighten enforcement on the safety gate (currently blocking).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a prompt → image progresses `GENERATING` → `SCORING` → `APPROVED` within the retry ceiling.
2. Force-fail rubric → image hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every attempt and every policy verdict for audit.
3. The safety gate blocks a description that contains a prohibited content signal, so the scorer never sees unsafe material.
4. Each completed cycle emits a `PolicyEvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
