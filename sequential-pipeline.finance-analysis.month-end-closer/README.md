# Akka Sample: Month-End Closer

A single `CloseAgent` walks a company's month-end through three task phases — **GATHER → VALIDATE → REPORT** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. An accounting manager reviews and approves each phase output before the pipeline advances. The user submits a close run and receives a signed `ClosePackage`.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: an application-level human-in-the-loop gate that pauses the workflow after each step pending accounting approval, and an `on-decision-eval` evaluator that scores every emitted journal-entry set against the trial balance for reconciliation accuracy.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every gather / validate / report tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.finance-analysis.month-end-closer  ~/my-projects/month-end-closer
cd ~/my-projects/month-end-closer
```

(Optional) Edit `SPEC.md` to point at a different chart of accounts, a different model provider, or a richer set of validation rules.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CloseAgent** — one AutonomousAgent declaring three Task constants (`GATHER_LEDGER_DATA`, `VALIDATE_ENTRIES`, `WRITE_CLOSE_REPORT`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **CloseRunWorkflow** — runs `gatherStep → approvalGate(GATHER) → validateStep → approvalGate(VALIDATE) → reportStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `CloseRunEntity` before the next step starts. The two `approvalGate` steps pause the workflow and surface an approval prompt on the UI.
- **CloseRunEntity** — an EventSourcedEntity holding the per-close-run lifecycle (`LedgerDataGathered`, `StepApproved`, `StepRejected`, `EntriesValidated`, `CloseReportWritten`, `ReconciliationScored`).
- **GatherTools / ValidateTools / ReportTools** — three function-tool classes registered on the agent, one per phase. The application-level HITL gate enforces that each downstream step only runs after its upstream approval has been recorded on the entity.
- **ApprovalGate** — the workflow step that blocks until an accounting user POSTs an approval or rejection decision via the API. A rejection rolls the phase back and re-runs the step; an approval advances the workflow.
- **ReconciliationScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `CloseReportWritten` and emits a 1–5 reconciliation score.
- **CloseRunView + CloseRunEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded chart-of-accounts files under `src/main/resources/sample-data/ledger/*.json` to fit your demo audience.
- `SPEC.md §4` and `prompts/close-agent.md` — narrow the agent's role (e.g., restrict it to accruals only, to intercompany eliminations, to payroll entries) by tightening the system prompt and renaming the typed records (`LedgerSnapshot`, `JournalEntrySet`, `ClosePackage`).
- `SPEC.md §5` — extend the typed outputs with industry-specific fields. The HITL approval gate does not need editing — it checks recorded approval events, not field shapes.
- `eval-matrix.yaml` — wire a real trial-balance reconciliation evaluator (replace the deterministic stub with a numeric balance check against a live GL export) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a close run → `GATHER` runs → accounting approves → `VALIDATE` runs → accounting approves → `REPORT` runs → a signed `ClosePackage` lands in the UI within ~120 s. Every transition is visible in real time.
2. Accounting rejects the gather phase output → the workflow replays `gatherStep` → the agent reruns → the rerun result is presented for a second approval → the pipeline eventually completes correctly.
3. Every `ClosePackage` emitted has an on-decision reconciliation score visible on the same UI card; packages whose trial balance does not zero receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the GATHER task does not see the report instructions, and the REPORT task does not see raw ledger rows — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
