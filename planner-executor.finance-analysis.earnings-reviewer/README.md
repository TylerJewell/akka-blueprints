# Akka Sample: Earnings Reviewer

A ReviewPlanner reads an earnings transcript and the matching 10-Q filing, dispatches passages to specialist readers, proposes updates to an internal financial model, and flags thesis-relevant changes with a confidence score. Demonstrates the **planner-executor** coordination pattern applied to finance analysis with embedded governance on the output side.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (runs-out-of-the-box): **None.** The transcript and filing sources are seeded text fixtures inside the Akka service.

## Generate the system

```sh
cp -r ./planner-executor.finance-analysis.earnings-reviewer ~/my-projects/earnings-reviewer
cd ~/my-projects/earnings-reviewer
```

(Optional) Edit `SPEC.md` to change the system name, the covered tickers, the model-update vocabulary, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReviewPlannerAgent** — AutonomousAgent that maintains a review ledger (passages found, model updates proposed, flags raised) and decides which specialist runs next.
- **TranscriptReaderAgent** — AutonomousAgent that extracts management-commentary and Q&A passages from a transcript fixture.
- **FilingReaderAgent** — AutonomousAgent that extracts line-item changes from a 10-Q or 10-K fixture.
- **ModelUpdaterAgent** — AutonomousAgent that converts passages into typed `ModelUpdate` records against the internal model.
- **ThesisFlagAgent** — AutonomousAgent that classifies each candidate flag as `THESIS_CONFIRMING`, `THESIS_CHALLENGING`, or `NOT_MATERIAL` and assigns a confidence in `[0, 1]`.
- **ReviewWorkflow** — Workflow with a plan → read → update-model → flag → score → finalize loop.
- **ReviewEntity** — EventSourcedEntity holding one review's lifecycle, the ledger, and the final answer.
- **ModelEntity** — EventSourcedEntity holding the internal financial model per ticker; updated only after a review reaches `COMPLETED`.
- **ReviewView** — projection used by the UI.
- **ReviewEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded tickers, the periods covered, or the simulator cadence.
- `SPEC.md §5` — adjust the `ModelUpdate` vocabulary (e.g., add `segment_revenue` as a tracked metric).
- `prompts/review-planner.md` — narrow the planner to a single sector or strategy.
- `eval-matrix.yaml` — add a halt control if you want unsafe flags to stop the review outright.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a `(ticker, period)` pair — planner runs the loop, model updates land on `ModelEntity`, and flags appear on the review answer within ~3 minutes.
2. A flag with confidence below the floor is blocked by the output guardrail; the planner is asked to revise.
3. Every accepted flag carries an evidence-quality score from the eval-event evaluator; scores below the floor are surfaced in the UI.
4. The simulator drips a sample review every 90 seconds so the App UI is not empty when first loaded.

## License

Apache 2.0.
