# Akka Sample: Finance Close Reconciler

A single reconciliation agent reads a period-end trial balance, matches GL account pairs against expected balances, and flags variances as items requiring accountant review. The trial balance rides into the agent as a task attachment, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern wired with four governance mechanisms: a financial-data sanitizer that strips confidential account codes before any LLM call, a `before-tool-call` guardrail that validates every proposed GL write before execution, a human-in-the-loop sign-off gate where the accountant approves reconciliation results, and a CI attestation gate that enforces audit-trail completeness on every build.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the trial balance corpus lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.finance-analysis.finance-close-reconciler  ~/my-projects/finance-close-reconciler
cd ~/my-projects/finance-close-reconciler
```

(Optional) Edit `SPEC.md` to point at a different chart-of-accounts structure (e.g., switch from the seeded IFRS trial balance to a US GAAP or cost-centre-based balance).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReconciliationAgent** — an AutonomousAgent that accepts a trial balance as a task attachment and a list of reconciliation rules, then returns a typed `ReconciliationReport`.
- **ReconciliationWorkflow** — orchestrates sanitize-wait → reconcile → hitl-await → attest steps per submitted period.
- **PeriodEntity** — an EventSourcedEntity holding the per-period reconciliation lifecycle.
- **BalanceSanitizer** — a Consumer that subscribes to `PeriodSubmitted` events, masks internal account codes and removes confidential margin data, and emits `BalanceSanitized` back to the entity.
- **ReconciliationView + ReconciliationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded chart of accounts for your own (the JSONL file under `src/main/resources/sample-events/account-rules.jsonl` after generation).
- `SPEC.md §5` — extend `ReconciliationReport` with organisation-specific fields (e.g., `costCentre`, `entityCode`, `currencyPair`).
- `prompts/reconciliation-agent.md` — narrow the agent's role (a manufacturing deployer would constrain it to plant-level standard costs; an insurance deployer to premium reserve accounts).
- `eval-matrix.yaml` — wire a real masking library for the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. An accountant submits a trial balance → it is sanitized → reconciled → the report appears in the UI awaiting sign-off.
2. The agent proposes a GL write with an invalid account code → the `before-tool-call` guardrail blocks it → the agent retries → a valid write proposal lands.
3. A report is submitted for human sign-off; the accountant approves it; the workflow advances to attestation and completes.
4. Confidential margin fields submitted in the trial balance never appear in the LLM call log; only the masked form does.

## License

Apache 2.0.
