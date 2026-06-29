# Akka Sample: General Ledger Reconciler

A single `LedgerAgent` walks a reconciliation run through three task phases — **FETCH → RECONCILE → DRAFT** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The controller submits an account set and receives a validated `JournalEntry` ready for posting.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a `before-agent-response` guardrail that validates every drafted journal entry before it can be committed to the books of record, and an application-level HITL escalation that routes runs with material variances to a controller for approval before posting proceeds.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every fetch / reconcile / draft tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.finance-analysis.gl-reconciler  ~/my-projects/gl-reconciler
cd ~/my-projects/gl-reconciler
```

(Optional) Edit `SPEC.md` to point at a different chart of accounts, a different model provider, or a richer set of reconciliation tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **LedgerAgent** — one AutonomousAgent declaring three Task constants (`FETCH_ENTRIES`, `RECONCILE_ACCOUNTS`, `DRAFT_JOURNAL`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **ReconciliationWorkflow** — runs `fetchStep → reconcileStep → draftStep → validateStep`. Each step calls `runSingleTask` and writes the typed result back onto `LedgerReconciliationEntity` before the next step starts. When the `PostingGuardrail` flags the draft, the workflow records the validation failure event and the run stops before any write reaches the books of record.
- **LedgerReconciliationEntity** — an EventSourcedEntity holding the per-run lifecycle (`EntriesFetched`, `ReconciliationCompleted`, `JournalDrafted`, `ValidationPassed`, `EscalationRaised`, `JournalPosted`, `RunFailed`).
- **FetchTools / ReconcileTools / JournalTools** — three function-tool classes registered on the agent, one per phase. The `before-agent-response` guardrail enforces that the drafted journal balances before the agent's task result is accepted.
- **PostingGuardrail** — the runtime check that backs the audit-material write constraint. A journal entry whose debits do not equal credits, or whose account codes are not present in the registered chart of accounts, is rejected before the task result is recorded.
- **ControllerEscalationService** — application-level HITL logic. When any variance in the `ReconciliationResult` exceeds the material threshold, the workflow raises an escalation and waits for a controller's `approve` or `reject` command before proceeding to `draftStep`.
- **ReconciliationView + ReconciliationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded account set under `src/main/resources/sample-data/accounts/*.json` to fit your demo chart of accounts.
- `SPEC.md §4` and `prompts/ledger-agent.md` — narrow the agent's role (e.g., constrain it to a single fund family, a single currency, or a specific accounting standard) by tightening the system prompt.
- `SPEC.md §5` — extend the typed outputs (`LedgerSnapshot`, `ReconciliationResult`, `JournalEntry`) with fund-specific fields such as NAV per unit. The posting guardrail does not need editing — it checks balance and account-code membership, not field shapes.
- `eval-matrix.yaml` — wire a real balance-assertion engine (replace the deterministic stub with a call to your GL system's validation API) by editing the `G1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A controller submits an account set → `FETCH` runs → `RECONCILE` runs → `DRAFT` runs → a validated `JournalEntry` lands in the UI within ~60 s. Every transition is visible in real time.
2. The agent drafts a journal entry that does not balance (forced via the mock LLM) → the `before-agent-response` guardrail rejects the draft → the workflow records the validation failure event → the run transitions to `FAILED` with the reason surfaced in the UI.
3. A reconciliation run whose variance exceeds the material threshold routes to `PENDING_APPROVAL` → a controller approves from the UI → the draft resumes and the journal is posted.
4. Each task receives only its own typed inputs; the FETCH task does not see the draft instructions, and the DRAFT task does not see raw ledger entry fetches — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
