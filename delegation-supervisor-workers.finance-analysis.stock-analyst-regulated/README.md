# Akka Sample: Stock Analysis Team

A supervisor delegates stock research to worker agents — news search, SEC filing review, summary, and a regulated investment recommendation — then holds the recommendation for live compliance review.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One model-provider key option: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` set in your shell — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box; news and SEC filings are modeled in-process from sample files.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.finance-analysis.stock-analyst-regulated  ~/my-projects/stock-analyst-regulated
cd ~/my-projects/stock-analyst-regulated
```

(Optional) Edit `SPEC.md` — system name, model provider, or the sample tickers.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- A `AnalysisWorkflow` (supervisor) that delegates research and aggregates results.
- Four worker agents: `NewsResearchAgent`, `FilingsResearchAgent`, `SummaryAgent`, `RecommendationAgent`.
- An `AnalysisEntity` (event-sourced) plus an `InboundRequestQueue` entity.
- An `AnalysisView` (read model) streamed to the UI over SSE.
- A `RequestConsumer`, a `RequestSimulator` (drips canned tickers), and a `DriftMonitor` (samples recommendation drift).
- Two HTTP endpoints (API + static UI) and the 5-tab UI shell.

## Customise before generating

- `SPEC.md` Section 1 — system name and pitch.
- `SPEC.md` Section 11 — model provider and per-agent mock outputs.
- `prompts/*.md` — agent behavior and the recommendation disclaimer text.
- The sample tickers in the simulator (named in Section 11).

## What gets validated

- Submitting a ticker drives an analysis from `QUEUED` to `ISSUED_PENDING_REVIEW` with a recommendation that carries a disclaimer.
- The sector sanitizer scrubs fair-disclosure-sensitive phrasing before the recommendation is persisted.
- A compliance reviewer can clear or retract an issued recommendation live.
- The drift monitor flags a skewed recommendation distribution.

See `reference/user-journeys.md` for the full acceptance journeys.

## License

Apache 2.0.
