# Akka Sample: SK Document Generator

A writer agent produces a structured document on a given topic; a reviewer agent evaluates the draft against quality criteria and returns either `APPROVE` or `REVISE` with typed feedback; the two iterate until the reviewer approves or the loop reaches its retry ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The blueprint runs out of the box; the inbound request stream and the document surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.content-editorial.writer-reviewer-doc-gen  ~/my-projects/writer-reviewer-doc-gen
cd ~/my-projects/writer-reviewer-doc-gen
```

(Optional) Edit `SPEC.md` to change the document schema, the reviewer rubric, the word-count ceiling, or the retry cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WriterAgent** — AutonomousAgent that produces a structured document draft on a given topic, accepting prior reviewer feedback when revising.
- **ReviewerAgent** — AutonomousAgent that evaluates a draft against quality criteria (structure, accuracy signals, clarity, length), returns either `APPROVE` or `REVISE` with a typed `ReviewNotes` payload.
- **DocumentWorkflow** — Workflow that runs the write → guardrail → review → revise loop up to a configurable retry ceiling, transitions the document to `APPROVED` on reviewer sign-off or to `REJECTED_FINAL` when the ceiling is hit.
- **DocumentEntity** — EventSourcedEntity that holds the document lifecycle, every draft attempt, every review, and the final outcome.
- **RequestQueue** — EventSourcedEntity that logs each submission for replay and audit.
- **DocumentsView** — read-side projection that the UI lists and streams via SSE.
- **DocRequestConsumer** — Consumer that starts a workflow per inbound submission.
- **RequestSimulator** — TimedAction that drips a sample topic every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **DocumentEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Document` record fields (e.g., raise `maxAttempts`, lower the word-count ceiling).
- `prompts/writer.md` — narrow the style to a different document type (e.g., technical brief, meeting summary).
- `prompts/reviewer.md` — change the rubric (e.g., add a citation-count requirement).
- `eval-matrix.yaml` — tighten enforcement on the output guardrail (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a topic → document progresses `DRAFTING` → `REVIEWING` → `APPROVED` within the retry ceiling.
2. Force-fail rubric → document hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every draft and every review for audit.
3. The output guardrail blocks a draft whose word count exceeds the ceiling, so the reviewer never sees over-length material.
4. Each completed cycle emits an `EvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
