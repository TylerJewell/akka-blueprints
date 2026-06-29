# Akka Sample: Self-Correcting Extraction Agent with Memory

A document-extraction agent pulls structured fields from unstructured input; a scorer grades each extraction against memory-grounded ground truth; the two iterate until the extraction passes the scorer or the loop hits its budget cap. Window-buffer memory carries recent extraction attempts across calls; database memory persists verified field-value pairs so every subsequent run benefits from prior corrections. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The blueprint runs out of the box; the inbound document stream and the outbound extraction surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.ops-automation.self-correcting-extraction  ~/my-projects/self-correcting-extraction
cd ~/my-projects/self-correcting-extraction
```

(Optional) Edit `SPEC.md` to change the field schema, the scoring rubric, the per-job budget cap, or the memory retention window.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ExtractionAgent** — AutonomousAgent that reads a raw document and returns a structured `FieldMap`; on a correction call, it receives the scorer's feedback and the window-buffer context and produces a revised extraction.
- **ScorerAgent** — AutonomousAgent that grades an extraction against memory-grounded ground truth, returning either `PASS` or `CORRECT` with a typed `CorrectionNotes` payload and a confidence score.
- **ExtractionWorkflow** — Workflow that runs the extract → score → correct loop up to a configurable budget cap, transitions the job to `VERIFIED` on scorer approval or to `BUDGET_EXHAUSTED` when the cap is hit.
- **ExtractionJobEntity** — EventSourcedEntity that holds the job lifecycle, every attempt's field map, every scorer verdict, and the final outcome.
- **DocumentQueue** — EventSourcedEntity that logs each submitted document for replay and audit.
- **JobsView** — read-side projection that the UI lists and streams via SSE.
- **DocumentConsumer** — Consumer that starts a workflow per inbound submission.
- **DocumentSimulator** — TimedAction that drips a sample document every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **ExtractionEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the documents the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `ExtractionJob` record fields (e.g., raise `budgetCap`, add or remove extracted fields).
- `prompts/extraction-agent.md` — narrow the extraction target to a different document type (e.g., invoices, lab reports, résumés).
- `prompts/scorer-agent.md` — change the scoring rubric (e.g., add a numeric-range constraint on certain fields).
- `eval-matrix.yaml` — tighten enforcement on the budget cap (currently system-level).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a document → job progresses `EXTRACTING` → `SCORING` → `VERIFIED` within the budget cap.
2. Force-fail scoring → job hits `BUDGET_EXHAUSTED` after the configured number of attempts; the entity preserves every attempt and every verdict for audit.
3. The on-decision eval records a scored verdict for each attempt; the App UI shows the per-attempt timeline.
4. Memory recall: a field value confirmed in an earlier job appears in the scorer's ground-truth context for a later job on the same document type.

## License

Apache 2.0.
