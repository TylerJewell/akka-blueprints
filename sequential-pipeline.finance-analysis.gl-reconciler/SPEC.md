# SPEC — gl-reconciler

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** General Ledger Reconciler.
**One-line pitch:** A controller submits an account set; one `LedgerAgent` walks it through three task phases — **FETCH** ledger entries from the books of record, **RECONCILE** them against expected balances and compute NAV, **DRAFT** a balanced journal entry — with each phase gated on the prior phase's recorded output, a `before-agent-response` guardrail that validates the draft before any write reaches the books of record, and a HITL escalation path for material variances.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a finance-analysis domain. One `LedgerAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the FETCH task's typed output becomes the RECONCILE task's instruction context; the RECONCILE task's typed output becomes the DRAFT task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-agent-response` guardrail** sits between the agent and the workflow's acceptance of each task result. It inspects the agent's returned `JournalEntry` before the workflow writes it onto the entity. Journal entries are audit-material: a draft whose debits do not equal credits, or whose account codes are absent from the registered chart of accounts, is rejected before the task result is recorded. The rejection returns a structured error to the agent loop so the draft task can self-correct inside its iteration budget. The same hook ensures the books of record are never written with an unbalanced entry.
- An **application-level HITL escalation** runs immediately after `ReconciliationCompleted` lands, as part of `reconcileStep`. `MaterialVarianceChecker` compares each `Variance.delta` against the configured material threshold. If any variance exceeds the threshold, the workflow transitions the run to `PENDING_APPROVAL`, raises an escalation record, and waits for a controller to issue an `approve` or `reject` command before proceeding to `draftStep`. The HITL path is wired as an explicit workflow state — not a side-channel — so it is auditable.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs enforce the dependency contract, the `before-agent-response` guardrail enforces write safety at the point where financial data would be committed, and the HITL escalation enforces controller oversight for material exceptions.

## 3. User-facing flows

The controller opens the App UI tab.

1. The controller types an **account set identifier** into the input (or picks one of three seeded account sets — `Fund A — Q2 close`, `Multi-currency sweep — June`, `NAV recalculation — series B`).
2. The controller clicks **Run reconciliation**. The UI POSTs to `/api/reconciliations` and receives a `runId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `FETCHING` — the workflow has started `fetchStep` and the agent has been handed the FETCH task.
4. Within ~10–20 s the card reaches `FETCHED` — the typed `LedgerSnapshot` is visible in the card detail (a table of ledger entries with account, date, debit/credit). The agent's FETCH task returned; the workflow recorded `EntriesFetched` and advanced.
5. If any variance in the computed reconciliation exceeds the material threshold, the card transitions to `PENDING_APPROVAL`. The controller reviews the variance table and either approves or rejects from the UI. On approval the run proceeds; on rejection the card reaches `REJECTED`.
6. Otherwise the card transitions to `RECONCILING`. Within ~10–20 s more it reaches `RECONCILED` — the `ReconciliationResult` is visible (variance table, NAV line, pass/fail per account).
7. Within ~10–20 s more the card reaches `DRAFTING`, then `DRAFT_VALIDATED`. The right pane shows the balanced `JournalEntry` — journal lines (account, debit, credit, description), total debits, total credits, and a balance-check chip (✓ balanced).
8. The controller can submit another account set; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ReconciliationEndpoint` | `HttpEndpoint` | `/api/reconciliations/*` — submit, list, get, SSE, approve, reject; serves `/api/metadata/*`. | — | `LedgerReconciliationEntity`, `ReconciliationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `LedgerReconciliationEntity` | `EventSourcedEntity` | Per-run lifecycle: created → fetching → fetched → reconciling → reconciled → pending_approval → drafting → draft_validated → posted → failed / rejected. Source of truth. | `ReconciliationEndpoint`, `ReconciliationWorkflow` | `ReconciliationView` |
| `ReconciliationWorkflow` | `Workflow` | One workflow per runId. Steps: `fetchStep` → `reconcileStep` → `hitlStep` (conditional) → `draftStep` → `validateStep`. Each agent-calling step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `ReconciliationEndpoint` after `CREATED` | `LedgerAgent`, `LedgerReconciliationEntity` |
| `LedgerAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `LedgerTasks.java`: `FETCH_ENTRIES` → `LedgerSnapshot`, `RECONCILE_ACCOUNTS` → `ReconciliationResult`, `DRAFT_JOURNAL` → `JournalEntry`. Each task is registered with the phase-appropriate function tools. | invoked by `ReconciliationWorkflow` | returns typed results |
| `FetchTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `fetchLedgerEntries(accountSetId)` and `fetchAccountBalances(accountSetId)`. Reads from `src/main/resources/sample-data/accounts/*.json` for deterministic offline output. | called from FETCH task | returns `List<LedgerEntry>` / `List<AccountBalance>` |
| `ReconcileTools` | function-tools class | Implements `computeVariances(snapshot)` and `calculateNAV(snapshot)`. Pure in-memory computations against expected-balance fixtures. | called from RECONCILE task | returns `List<Variance>` / `NavCalculation` |
| `JournalTools` | function-tools class | Implements `formatJournalLine(variance)` and `sumJournalLines(lines)`. | called from DRAFT task | returns `JournalLine` / `JournalTotals` |
| `PostingGuardrail` | `before-agent-response` guardrail (registered on `LedgerAgent`) | Inspects the agent's returned `JournalEntry`. Rejects any draft whose `totalDebits != totalCredits` or whose `lines[*].accountId` contains a code absent from the registered chart of accounts. | every agent task result on every task | accept / structured-reject |
| `MaterialVarianceChecker` | plain class (no Akka primitive) | Pure deterministic check. Inputs: `ReconciliationResult`, material threshold. Output: `EscalationDecision{escalate, reason}`. | called from `reconcileStep` | returns decision |
| `ReconciliationView` | `View` | Read model: one row per reconciliation run for the UI. | `LedgerReconciliationEntity` events | `ReconciliationEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record LedgerEntry(
    String entryId,
    String accountId,
    String description,
    BigDecimal debit,
    BigDecimal credit,
    LocalDate postingDate
) {}

record AccountBalance(
    String accountId,
    String accountName,
    BigDecimal expectedBalance,
    String currency
) {}

record LedgerSnapshot(
    String accountSetId,
    List<LedgerEntry> entries,
    List<AccountBalance> expectedBalances,
    Instant fetchedAt
) {}

record Variance(
    String accountId,
    BigDecimal expectedBalance,
    BigDecimal actualBalance,
    BigDecimal delta,
    String currency,
    boolean isMaterial
) {}

record NavCalculation(
    String fundId,
    BigDecimal totalAssets,
    BigDecimal totalLiabilities,
    BigDecimal nav,
    Instant calculatedAt
) {}

record ReconciliationResult(
    List<Variance> variances,
    Optional<NavCalculation> nav,
    boolean allAccountsReconciled,
    boolean hasMaterialVariance,
    Instant reconciledAt
) {}

record JournalLine(
    String lineId,
    String accountId,
    BigDecimal debit,
    BigDecimal credit,
    String description,
    String reference
) {}

record JournalEntry(
    String journalId,
    List<JournalLine> lines,
    BigDecimal totalDebits,
    BigDecimal totalCredits,
    LocalDate periodEnd,
    Instant draftedAt
) {}

record ValidationResult(
    boolean passed,
    List<String> findings,
    Instant validatedAt
) {}

record EscalationRecord(
    String escalationId,
    String runId,
    String reason,
    Optional<String> reviewedBy,
    Optional<String> decision,
    Instant raisedAt,
    Optional<Instant> decidedAt
) {}

record ReconciliationRun(
    String runId,
    Optional<String> accountSetId,
    Optional<LedgerSnapshot> snapshot,
    Optional<ReconciliationResult> reconciliation,
    Optional<EscalationRecord> escalation,
    Optional<JournalEntry> journal,
    Optional<ValidationResult> validation,
    ReconciliationStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReconciliationStatus {
    CREATED, FETCHING, FETCHED, RECONCILING, RECONCILED,
    PENDING_APPROVAL, DRAFTING, DRAFT_VALIDATED, POSTED,
    REJECTED, FAILED
}
```

Events on `LedgerReconciliationEntity`: `RunCreated`, `FetchStarted`, `EntriesFetched`, `ReconcileStarted`, `ReconciliationCompleted`, `EscalationRaised`, `EscalationApproved`, `EscalationRejected`, `DraftStarted`, `JournalDrafted`, `ValidationPassed`, `ValidationFailed`, `JournalPosted`, `RunFailed`.

Every nullable lifecycle field on the `ReconciliationRun` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/reconciliations` — body `{ accountSetId }` → `{ runId }`.
- `GET /api/reconciliations` — list all runs, newest-first.
- `GET /api/reconciliations/{id}` — one run.
- `GET /api/reconciliations/sse` — Server-Sent Events; one event per state transition.
- `POST /api/reconciliations/{id}/approve` — body `{ reviewedBy }` → approves a `PENDING_APPROVAL` run.
- `POST /api/reconciliations/{id}/reject` — body `{ reviewedBy, reason }` → rejects a `PENDING_APPROVAL` run.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: General Ledger Reconciler</title>`.

The App UI tab is a two-column layout: a left rail with the live list of reconciliation runs (status pill + account set + age) and a right pane with the selected run's detail — account set, fetched entries table, variance table, NAV line, journal entry lines, validation chip, escalation strip if the run required controller approval, and a failure-reason strip if the run failed.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (posting-safety gate)**: `PostingGuardrail` is registered on `LedgerAgent` and runs before the workflow accepts each task result. It inspects the agent's returned `JournalEntry` and applies two rules: (1) balance rule — `totalDebits == totalCredits`; (2) account-code rule — every `JournalLine.accountId` exists in the chart of accounts loaded from `src/main/resources/sample-data/coa.json`. If either rule fails, the guardrail rejects the result with a structured `posting-safety-violation` error to the agent loop and the workflow records a `ValidationFailed{findings}` event. The agent loop retries within its 4-iteration budget. Because journal entries are audit-material, the guardrail is the last line of defence before any data would reach the books of record — it fires even if prior tasks completed cleanly.
- **H1 — application HITL (controller escalation)**: runs immediately after `ReconciliationCompleted` lands, inside `reconcileStep`. `MaterialVarianceChecker` compares each `Variance.delta.abs()` against the configured material threshold (default: 1 % of `AccountBalance.expectedBalance`). If any variance is material, the workflow writes `EscalationRaised{reason}` onto the entity, sets status to `PENDING_APPROVAL`, and waits. The controller reviews the variance table in the App UI and POSTs to `/api/reconciliations/{id}/approve` or `/api/reconciliations/{id}/reject`. On approval the workflow resumes `draftStep`; on rejection the workflow writes `EscalationRejected` and ends at `REJECTED`. The HITL path is an explicit workflow state transition — not a notification side-channel — so every escalation decision is recorded in the entity log.

## 9. Agent prompts

- `LedgerAgent` → `prompts/ledger-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Controller submits the seeded account set `Fund A — Q2 close`; within 60 s the run reaches `DRAFT_VALIDATED` with non-empty entries, ≥ 1 variance row, a balanced journal, and a balance-check chip showing ✓.
2. **J2** — The agent's DRAFT task returns an unbalanced `JournalEntry` (mock LLM path). `PostingGuardrail` rejects the result; a `ValidationFailed` event lands on the entity; the agent retries, returns a balanced draft; the run completes correctly. The UI's failure strip shows the one rejected draft.
3. **J3** — The reconciliation for `Multi-currency sweep — June` produces a material variance. The run transitions to `PENDING_APPROVAL`. The controller approves from the UI. The run proceeds to `draftStep` and completes as in J1.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-run trace (logged at `INFO`); the FETCH task's log shows only FETCH-tool calls, the RECONCILE task's log shows only RECONCILE-tool calls, the DRAFT task's log shows only DRAFT-tool calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named gl-reconciler demonstrating the sequential-pipeline x finance-analysis cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-finance-analysis-gl-reconciler. Java package
io.akka.samples.generalledgerreconciler. Akka 3.6.0. HTTP port 9371.

Components to wire (exactly):

- 1 AutonomousAgent LedgerAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/ledger-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  FETCH, RECONCILE, and DRAFT tool sets are ALL registered on the agent; phase gating and
  response validation are the job of PostingGuardrail, NOT of conditional .tools(...) wiring.
  The before-agent-response guardrail (PostingGuardrail) is registered on the agent via the
  agent's guardrail-configuration block. On guardrail rejection the agent loop retries within
  its 4-iteration budget.

- 1 Workflow ReconciliationWorkflow per runId with five steps:
  * fetchStep — emits FetchStarted on the entity, then calls componentClient
    .forAutonomousAgent(LedgerAgent.class, "agent-" + runId).runSingleTask(
      TaskDef.instructions("AccountSet: " + accountSetId + "\nPhase: FETCH\nUse
      fetchLedgerEntries and fetchAccountBalances to collect the full ledger snapshot for
      this account set.")
        .metadata("runId", runId)
        .metadata("phase", "FETCH")
        .taskType(LedgerTasks.FETCH_ENTRIES)
    ). Reads forTask(taskId).result(FETCH_ENTRIES) to get LedgerSnapshot. Writes
    LedgerReconciliationEntity.recordSnapshot(snapshot). WorkflowSettings.stepTimeout 60s.
  * reconcileStep — emits ReconcileStarted, then runSingleTask with TaskDef.instructions
    (formatReconcileContext(snapshot, accountSetId)) and metadata.phase = "RECONCILE", taskType
    RECONCILE_ACCOUNTS. Writes LedgerReconciliationEntity.recordReconciliation(result).
    Then calls MaterialVarianceChecker.check(result, threshold). If escalate=true, emits
    EscalationRaised and transitions to PENDING_APPROVAL, pausing the workflow. stepTimeout 60s.
  * hitlStep — waits for controller approve/reject command. On EscalationApproved, emits
    EscalationApproved and advances to draftStep. On EscalationRejected, emits
    EscalationRejected, writes RunFailed, and ends. stepTimeout 72h (controller has time
    to review).
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions
    (formatDraftContext(reconciliation, snapshot, accountSetId)) and metadata.phase = "DRAFT",
    taskType DRAFT_JOURNAL. Writes LedgerReconciliationEntity.recordJournal(journal).
    stepTimeout 60s.
  * validateStep — the PostingGuardrail already fired before draftStep returned; if the
    workflow reaches validateStep, the journal passed. Emits ValidationPassed and
    JournalPosted, ends in POSTED. stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(ReconciliationWorkflow::error). The error step writes
  RunFailed and ends.

- 1 EventSourcedEntity LedgerReconciliationEntity (one per runId). State ReconciliationRun{runId,
  accountSetId: Optional<String>, snapshot: Optional<LedgerSnapshot>,
  reconciliation: Optional<ReconciliationResult>, escalation: Optional<EscalationRecord>,
  journal: Optional<JournalEntry>, validation: Optional<ValidationResult>,
  status: ReconciliationStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  ReconciliationStatus enum: CREATED, FETCHING, FETCHED, RECONCILING, RECONCILED,
  PENDING_APPROVAL, DRAFTING, DRAFT_VALIDATED, POSTED, REJECTED, FAILED.
  Events: RunCreated{accountSetId}, FetchStarted, EntriesFetched{snapshot},
  ReconcileStarted, ReconciliationCompleted{reconciliation}, EscalationRaised{reason},
  EscalationApproved{reviewedBy}, EscalationRejected{reviewedBy, reason},
  DraftStarted, JournalDrafted{journal}, ValidationPassed{validatedAt},
  ValidationFailed{findings}, JournalPosted{postedAt}, RunFailed{reason}.
  Commands: create, startFetch, recordSnapshot, startReconcile, recordReconciliation,
  raiseEscalation, approveEscalation, rejectEscalation, startDraft, recordJournal,
  recordValidationPassed, recordValidationFailed, post, fail, getRun.
  emptyState() returns ReconciliationRun.initial("") with all Optional fields as
  Optional.empty() and no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View ReconciliationView with row type ReconciliationRow that mirrors ReconciliationRun
  exactly (all Optional<T> lifecycle fields preserved). Table updater consumes
  LedgerReconciliationEntity events. ONE query getAllRuns: SELECT * AS runs FROM
  reconciliation_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ReconciliationEndpoint at /api with POST /reconciliations (body {accountSetId}; mints
    runId; calls LedgerReconciliationEntity.create(accountSetId); then starts
    ReconciliationWorkflow with id "recon-" + runId; returns {runId}), GET /reconciliations
    (list from getAllRuns, sorted newest-first), GET /reconciliations/{id} (one row),
    GET /reconciliations/sse (Server-Sent Events forwarded from the view's stream-updates),
    POST /reconciliations/{id}/approve (body {reviewedBy}; calls approveEscalation),
    POST /reconciliations/{id}/reject (body {reviewedBy, reason}; calls rejectEscalation),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- LedgerTasks.java declaring three Task<R> constants:
    FETCH_ENTRIES = Task.name("Fetch entries").description("Retrieve all ledger entries and
      expected account balances for an account set by calling fetchLedgerEntries and
      fetchAccountBalances").resultConformsTo(LedgerSnapshot.class);
    RECONCILE_ACCOUNTS = Task.name("Reconcile accounts").description("Compute per-account
      variances and calculate NAV by calling computeVariances and calculateNAV").
      resultConformsTo(ReconciliationResult.class);
    DRAFT_JOURNAL = Task.name("Draft journal").description("Compose a balanced journal entry
      whose lines offset each material variance").resultConformsTo(JournalEntry.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {FETCH, RECONCILE, DRAFT}. Each function-tool method is annotated with
  the constant phase, e.g. @FunctionTool(name = "fetchLedgerEntries", phase = Phase.FETCH)
  (use a custom annotation if the SDK's @FunctionTool does not carry a phase field — the
  guardrail reads it from a parallel registry built at startup if so).

- FetchTools.java — @FunctionTool fetchLedgerEntries(String accountSetId) ->
  List<LedgerEntry> reading from src/main/resources/sample-data/accounts/<accountSetId>.json;
  @FunctionTool fetchAccountBalances(String accountSetId) -> List<AccountBalance> reading
  expected balances from the same file.

- ReconcileTools.java — @FunctionTool computeVariances(LedgerSnapshot) -> List<Variance>
  (one Variance per AccountBalance entry, delta = actualBalance - expectedBalance, isMaterial
  = abs(delta) > threshold); @FunctionTool calculateNAV(LedgerSnapshot) -> NavCalculation
  (NAV = totalAssets - totalLiabilities, values derived from the snapshot's debit/credit sums
  per account type).

- JournalTools.java — @FunctionTool formatJournalLine(Variance) -> JournalLine (lineId minted
  as "jl-" + accountId, debit/credit set to offset the delta); @FunctionTool
  sumJournalLines(List<JournalLine>) -> JournalTotals (sum of all debit and credit amounts).

- PostingGuardrail.java — implements the before-agent-response hook. Inspects the agent's
  returned JournalEntry. Applies two rules: (1) totalDebits.compareTo(totalCredits) == 0;
  (2) every lines[i].accountId is present in a ChartOfAccounts loaded at startup from
  src/main/resources/sample-data/coa.json. On rule failure, returns
  Guardrail.reject("posting-safety-violation: <reason>"). On rejection ALSO calls
  LedgerReconciliationEntity.recordValidationFailed(findings) so the failure is visible
  in the UI.

- MaterialVarianceChecker.java — pure deterministic logic (no LLM). Inputs:
  ReconciliationResult, BigDecimal materialThreshold. Outputs: EscalationDecision{escalate,
  reason}. Checks each Variance: if any isMaterial == true, returns escalate=true with a
  reason naming all material accounts. Called from reconcileStep AFTER the task result is
  recorded.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9371 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-data/accounts/*.json — three files keyed by accountSetId
  (fund-a-q2-close.json, multi-currency-sweep-june.json, nav-recalculation-series-b.json),
  each carrying 6-10 LedgerEntry items and matching AccountBalance items. The
  multi-currency-sweep-june.json file includes one AccountBalance whose actual entries sum to
  a delta exceeding 1 % threshold so J3 (material variance escalation) is reproducible.

- src/main/resources/sample-data/coa.json — a ChartOfAccounts object listing all valid
  account codes referenced in the sample ledger files, used by PostingGuardrail.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, H1) matching the mechanisms in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root pre-filled for the finance-analysis domain.

- prompts/ledger-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: General Ledger Reconciler",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of run cards; right = selected-run detail with account set header, entries table,
  variance table, NAV line, journal-lines table, balance-check chip, escalation strip,
  validation-failure strip). Browser title exactly: <title>Akka Sample: General Ledger
  Reconciler</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(runId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    fetch-entries.json — 6 LedgerSnapshot entries keyed by seeded account set. Each entry's
      tool_calls array contains 2 calls: fetchLedgerEntries + fetchAccountBalances.
    reconcile-accounts.json — 6 ReconciliationResult entries paired one-to-one with the
      fetch entries, each with 2-4 Variance items. The multi-currency-sweep-june entry
      carries at least one isMaterial=true variance to exercise J3.
      tool_calls contain computeVariances + calculateNAV in order.
    draft-journal.json — 6 JournalEntry entries paired one-to-one with reconciliation
      entries, each with balanced debit/credit totals. tool_calls contain formatJournalLine
      (one per variance) + sumJournalLines. PLUS 1 deliberately UNBALANCED entry whose
      totalDebits != totalCredits — selected on the FIRST iteration of every 4th run
      (modulo seed) so J2 (guardrail rejection) is reproducible. The mock then falls through
      to a balanced retry on the next iteration.
- A MockModelProvider.seedFor(runId) helper makes per-run selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. LedgerAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion LedgerTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (fetchStep
  60s, reconcileStep 60s, hitlStep 72h, draftStep 60s, validateStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the ReconciliationRun row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: LedgerTasks.java with FETCH_ENTRIES, RECONCILE_ACCOUNTS, DRAFT_JOURNAL constants
  is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9371 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (LedgerAgent). The
  HITL check is deterministic (MaterialVarianceChecker.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  the before-agent-response guardrail (PostingGuardrail) is the runtime mechanism that
  enforces posting safety. Do NOT conditionally register tools per task — the guardrail is
  the gate.
- Task dependency is carried by typed task results: fetchStep writes LedgerSnapshot onto the
  entity, reconcileStep reads it and builds the RECONCILE task's instruction context from it,
  draftStep reads both. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
