# Akka Sample: Adaptive Practice Coach

A tutor agent generates practice items targeted to a learner's current knowledge level; a scorer agent evaluates the learner's response and reports diagnostic metrics; a mastery workflow adjusts difficulty for the next item based on those metrics. Demonstrates the **evaluator-optimizer** coordination pattern applied to adaptive learning.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The blueprint runs out of the box; all item generation, scoring, and mastery tracking run inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.education.adaptive-practice-coach  ~/my-projects/adaptive-practice-coach
cd ~/my-projects/adaptive-practice-coach
```

(Optional) Edit `SPEC.md` to change the subject domain, the mastery threshold, the difficulty scale, or the session item cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ItemGeneratorAgent** — AutonomousAgent that produces a practice item (question + answer key) for a given topic and difficulty level, incorporating any prior scorer feedback on the last item's diagnostic gaps.
- **ScorerAgent** — AutonomousAgent that evaluates a learner's response against the answer key and returns a typed `ScoreReport` with correctness, partial-credit analysis, and a diagnostic tag.
- **PracticeSessionWorkflow** — Workflow that manages an item-generate → curriculum-check → learner-response → score → adapt loop up to a configurable session item cap, with mastery detection ending the session early.
- **LearnerSessionEntity** — EventSourcedEntity that holds the full session: every item, every response, every score report, the running mastery estimate, and the terminal outcome.
- **EnrollmentQueue** — EventSourcedEntity that logs each session-start request for replay and audit.
- **SessionsView** — read-side projection that the UI lists and streams via SSE.
- **SessionRequestConsumer** — Consumer that starts a workflow per inbound enrollment event.
- **EnrollmentSimulator** — TimedAction that drips a sample learner enrollment every 60 s so the App UI is never empty.
- **DiagnosticSampler** — TimedAction that records a `DiagnosticRecorded` event for every scored item that has not yet been sampled.
- **CoachEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `LearnerSession` record fields (e.g., raise `masteryThreshold`, change the difficulty scale).
- `prompts/item-generator.md` — narrow the subject domain (e.g., algebra, world history).
- `prompts/scorer.md` — change the scoring rubric (e.g., add a reasoning-quality dimension).
- `eval-matrix.yaml` — tighten enforcement on the curriculum-alignment guardrail (currently blocking).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Start a session → the learner answers items, the mastery estimate rises, the session ends with `MASTERED` before the item cap.
2. Learner consistently struggles → session runs to the item cap and ends with `CAP_REACHED`; every item and score is preserved for audit.
3. The curriculum-alignment guardrail blocks an off-topic item before the learner sees it; the generator produces a replacement.
4. Each completed score emits a `DiagnosticRecorded` event that surfaces in the App UI's per-item timeline.

## License

Apache 2.0.
