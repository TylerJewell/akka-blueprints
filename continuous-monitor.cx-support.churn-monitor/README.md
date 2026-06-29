# Akka Sample: Customer Churn Prediction Agent (Partner)

A continuous background worker scores customer accounts for churn risk, strips PII from customer profiles before passing them to an AI scoring model, and surfaces prioritised retention recommendations for a customer-success team. Demonstrates the **continuous-monitor** coordination pattern wired with two governance mechanisms (PII sanitizer, periodic drift + fairness eval).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./continuous-monitor.cx-support.churn-monitor  ~/my-projects/churn-monitor
cd ~/my-projects/churn-monitor
```

(Optional) Edit `SPEC.md` to point `AccountPoller` at a real CRM data source or to keep the in-memory account simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AccountPoller** — TimedAction firing every 60 s that drips simulated account snapshots into `AccountSnapshotQueue`.
- **PiiSanitizer** — Consumer that redacts PII fields from account snapshots before the LLM sees them.
- **ChurnScoringAgent** — Agent (typed) that produces a `ChurnScore` (risk level, probability, top risk signals).
- **RetentionAdvisorAgent** — AutonomousAgent that generates a ranked `RetentionPlan` for high-risk accounts.
- **ChurnWorkflow** — Workflow orchestrating: sanitize → score → (if high risk) advise → surface for review.
- **AccountChurnEntity** — EventSourcedEntity holding each account's churn lifecycle.
- **ChurnView + ChurnEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **DriftFairnessEvalRunner** — TimedAction running every 6 hours; samples scored accounts for model drift and demographic bias.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated account feed for a real CRM source.
- `SPEC.md §5` — extend `AccountSnapshot` with org-specific fields (`contractValue`, `supportTier`, `npsScore`, etc.).
- `prompts/churn-scoring.md` — tune risk-signal weighting for your product vertical.
- `prompts/retention-advisor.md` — constrain the retention playbook to offers your team can actually deliver.
- `eval-matrix.yaml` — wire a real fairness library by adding it under the drift-fairness-watch implementation.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated account snapshot arrives → it is sanitized → scored → (if high risk) a retention plan is produced → the account appears in the UI within 60 s.
2. A customer-success manager marks a retention recommendation as actioned — the account transitions to ACTIONED.
3. A manager dismisses a recommendation with a reason — the account transitions to DISMISSED.
4. DriftFairnessEvalRunner runs and surfaces a drift alert on accounts whose score distribution has shifted.

## License

Apache 2.0.
