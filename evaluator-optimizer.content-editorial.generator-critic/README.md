# Akka Sample: Basic Reflection

A generator agent writes an essay on a given topic; a reflector agent critiques the writing and returns structured feedback; the generator revises in a fixed-round loop until the reflector accepts or the round ceiling is reached. A policy guardrail gates the final draft before it is released to the caller. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the inbound request stream and the outbound content surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.content-editorial.generator-critic  ~/my-projects/generator-critic
cd ~/my-projects/generator-critic
```

(Optional) Edit `SPEC.md` to change the essay style brief, the acceptance rubric, the word count ceiling, or the round cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GeneratorAgent** — AutonomousAgent that writes an essay draft on the submitted topic, incorporating reflector feedback on revision calls.
- **ReflectorAgent** — AutonomousAgent that scores a draft against coherence, clarity, and topic adherence criteria, returning either `ACCEPT` or `REVISE` with a typed `ReflectionNotes` payload.
- **ReflectionWorkflow** — Workflow that runs the generate → reflect → revise loop up to a configurable round ceiling, then transitions the essay to `ACCEPTED` on reflector approval or to `REJECTED_FINAL` when the ceiling is hit.
- **EssayEntity** — EventSourcedEntity holding the essay lifecycle, every round's draft, every reflection, and the final outcome.
- **SubmissionQueue** — EventSourcedEntity logging each submission for replay and audit.
- **EssaysView** — read-side projection that the UI lists and streams via SSE.
- **SubmissionConsumer** — Consumer that starts a workflow per inbound submission.
- **TopicSimulator** — TimedAction that drips a sample topic every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-round eval event each cycle (control E1).
- **EssayEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Essay` record fields (e.g., raise `maxRounds`, change the word-count ceiling).
- `prompts/generator.md` — narrow the writing style or target audience.
- `prompts/reflector.md` — change the acceptance rubric (e.g., add a citation-quality criterion).
- `eval-matrix.yaml` — adjust enforcement on the output guardrail (currently blocking) or the eval-event recording (currently non-blocking).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a topic → essay progresses `DRAFTING` → `REFLECTING` → `ACCEPTED` within the round ceiling.
2. Force-fail rubric → essay hits `REJECTED_FINAL` after the configured number of rounds; the entity preserves every round and every reflection for audit.
3. The output guardrail blocks the final accepted draft from release if it violates the policy check; the draft is revised before release.
4. Each completed round emits a `ReflectionRecorded` event that surfaces in the App UI's per-round timeline.

## License

Apache 2.0.
