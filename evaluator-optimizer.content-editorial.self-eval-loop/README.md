# Akka Sample: Self-Eval Loop Posts

A poet agent drafts a Shakespearean-style short post; a critic agent scores it against length and style rules; the two pass feedback back and forth until the critic accepts the draft or the loop hits its retry ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The blueprint runs out of the box; the inbound request stream and the outbound posting surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.content-editorial.self-eval-loop  ~/my-projects/self-eval-loop-posts
cd ~/my-projects/self-eval-loop-posts
```

(Optional) Edit `SPEC.md` to change the style brief, the acceptance rubric, the per-post character ceiling, or the retry cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PoetAgent** — AutonomousAgent that drafts a Shakespearean-style post under a character ceiling, accepting prior critic feedback when revising.
- **CriticAgent** — AutonomousAgent that scores a draft against length and style rules, returns either `ACCEPT` or `REVISE` with a typed `CritiqueNotes` payload.
- **RefinementWorkflow** — Workflow that runs the draft → critique → revise loop up to a configurable retry ceiling, transitions the post to `ACCEPTED` on critic approval or to `REJECTED_FINAL` when the ceiling is hit.
- **PostEntity** — EventSourcedEntity that holds the post lifecycle, every attempt's draft, every critique, and the final outcome.
- **RequestQueue** — EventSourcedEntity that logs each submission for replay and audit.
- **PostsView** — read-side projection that the UI lists and streams via SSE.
- **DraftRequestConsumer** — Consumer that starts a workflow per inbound submission.
- **RequestSimulator** — TimedAction that drips a sample brief every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **PoetryEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the briefs the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Post` record fields (e.g., raise `maxAttempts`, lower the per-post character ceiling).
- `prompts/poet.md` — narrow the style to a different period or register (e.g., haiku, Edwardian formal).
- `prompts/critic.md` — change the rubric (e.g., add a meter constraint).
- `eval-matrix.yaml` — tighten enforcement on the output guardrail (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a brief → post progresses `DRAFTING` → `EVALUATING` → `ACCEPTED` within the retry ceiling.
2. Force-fail rubric → post hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every attempt and every critique for audit.
3. The output guardrail blocks a draft whose character count exceeds the ceiling, so the critic never sees over-length material.
4. Each completed cycle emits an `EvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
