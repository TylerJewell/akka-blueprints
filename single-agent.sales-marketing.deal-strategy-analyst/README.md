# Akka Sample: Deal Strategy Analyst

A single deal-strategy agent ingests scattered deal context — emails, call notes, CRM records — and produces a prioritised set of recommended next steps: which stakeholder to contact, what concern to address, and what action to take by when. The deal context rides into the agent as a task attachment, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern in the sales-marketing domain. One `DealStrategyAgent` (AutonomousAgent) makes the recommendation; the surrounding components prepare its input, filter confidential material, and score the output for completeness.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — deal context lives in-process and external CRM data is seeded.

## Generate the system

```sh
cp -r ./single-agent.sales-marketing.deal-strategy-analyst  ~/my-projects/deal-strategy
cd ~/my-projects/deal-strategy
```

(Optional) Edit `SPEC.md` to adjust the seeded deal scenarios (e.g., change the deal stage vocabulary or add a custom next-step category).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DealStrategyAgent** — an AutonomousAgent that accepts a deal context bundle (emails, call notes, CRM snapshot) as a task attachment and returns a typed `StrategyRecommendation`.
- **AnalysisWorkflow** — orchestrates sanitize-wait → analyse → eval per submitted deal.
- **DealEntity** — an EventSourcedEntity holding the per-deal analysis lifecycle.
- **DealContextSanitizer** — a Consumer that subscribes to `DealSubmitted` events, redacts PII and NDA-sensitive names into a `SanitizedContext`, and emits `ContextSanitized` back to the entity.
- **DealView + DealEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded deal scenarios (the JSONL file under `src/main/resources/sample-events/seed-deals.jsonl` after generation).
- `SPEC.md §5` — extend `StrategyRecommendation` with pipeline-specific fields (e.g., `dealStage`, `forecastCategory`, `estimatedCloseDate`).
- `prompts/deal-strategy-analyst.md` — narrow the agent's focus (e.g., constrain it to multi-stakeholder enterprise deals or shorten the recommendation horizon).
- `eval-matrix.yaml` — wire a real PII redactor by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a deal context bundle → it is sanitized → analysed → the recommendation appears in the UI.
2. The agent returns a malformed recommendation on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed recommendation lands.
3. Every recorded recommendation has an on-decision eval score visible on the same UI card.
4. Buyer email addresses and company names in the raw context never appear in the LLM call log; only redacted tokens do.

## License

Apache 2.0.
