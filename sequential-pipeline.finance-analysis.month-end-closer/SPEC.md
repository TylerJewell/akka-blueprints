# SPEC — month-end-closer

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Month-End Closer.
**One-line pitch:** A user submits a close run; one `CloseAgent` walks it through three task phases — **GATHER** ledger data, **VALIDATE** journal entries against the trial balance, **WRITE** a signed close report — with each phase gated on accounting approval of the prior phase's output and each phase's tools rejected when called out of order.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a finance-analysis domain. One `CloseAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the GATHER task's typed output becomes the VALIDATE task's instruction context; the VALIDATE task's typed output becomes the WRITE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- An **application-level human-in-the-loop gate** sits between each phase's completion and the next phase's start. After `gatherStep` records `LedgerDataGathered`, the workflow enters an `approvalGate(GATHER)` step that blocks until an accounting user explicitly approves or rejects the phase output via the API. A rejection causes the workflow to replay the gather step; an approval advances to `validateStep`. The same pattern applies between `validateStep` and `reportStep`. The approval decision, approver identity, and timestamp are recorded on `CloseRunEntity` as `StepApproved` or `StepRejected` events for audit.
- An **`on-decision-eval`** runs immediately after `CloseReportWritten` lands, as `evalStep` inside the workflow. A deterministic, rule-based `ReconciliationScorer` (no LLM call — the eval is rule-based on purpose, so the same close package always scores the same) checks that debits equal credits across all journal entries, that every journal entry references a valid account code from the chart of accounts, that every accrual entry has a matching reversal date, and that the net trial-balance variance is within the configured materiality threshold.

The blueprint shows that a sequential pipeline in a regulated finance domain is not just a chain of LLM calls — the task-boundary handoffs enforce the dependency contract, the HITL gates enforce human accountability, and the eval enforces reconciliation accuracy.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **period** (e.g. `2026-05`) and a **legal entity** from the dropdown (or types a custom entity code), then clicks **Start close run**. The UI POSTs to `/api/close-runs` and receives a `closeRunId`.
2. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `GATHERING` — the workflow has started `gatherStep` and the agent has been handed the GATHER task.
3. Within ~20–30 s the card reaches `AWAITING_GATHER_APPROVAL`. The right pane shows the gathered `LedgerSnapshot` — a table of ledger lines with account code, description, debit amount, credit amount, and posting date. A yellow **Approve** / **Reject** button pair appears for accounting users.
4. The accounting user clicks **Approve gather**. The card transitions to `VALIDATING`. Within ~20–30 s more the card reaches `AWAITING_VALIDATE_APPROVAL`. The right pane shows the `JournalEntrySet` — a list of proposed journal entries with debit/credit pairs, account codes, and entry narrations.
5. The accounting user clicks **Approve validate**. The card transitions to `REPORTING`. Within ~20–30 s more the card reaches `EVALUATED`. The right pane shows the full `ClosePackage` — a signed close report with a trial-balance summary, journal-entry list, and material-variance commentary — plus a reconciliation score chip (1–5) and a one-line rationale.
6. The user can start another close run; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CloseRunEndpoint` | `HttpEndpoint` | `/api/close-runs/*` — submit, list, get, approve/reject, SSE; serves `/api/metadata/*`. | — | `CloseRunEntity`, `CloseRunView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `CloseRunEntity` | `EventSourcedEntity` | Per-close-run lifecycle: created → gathering → awaiting-gather-approval → validating → awaiting-validate-approval → reporting → evaluated (or failed / rejected). Source of truth. | `CloseRunEndpoint`, `CloseRunWorkflow` | `CloseRunView` |
| `CloseRunWorkflow` | `Workflow` | One workflow per close run. Steps: `gatherStep` → `approvalGate(GATHER)` → `validateStep` → `approvalGate(VALIDATE)` → `reportStep` → `evalStep`. Each agent-calling step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then parks at the approval gate. | started by `CloseRunEndpoint` after `CREATED` | `CloseAgent`, `CloseRunEntity` |
| `CloseAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `CloseTasks.java`: `GATHER_LEDGER_DATA` → `LedgerSnapshot`, `VALIDATE_ENTRIES` → `JournalEntrySet`, `WRITE_CLOSE_REPORT` → `ClosePackage`. Each task is registered with the phase-appropriate function tools. | invoked by `CloseRunWorkflow` | returns typed results |
| `GatherTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `fetchLedgerLines(entity, period)` and `fetchChartOfAccounts(entity)`. Reads from `src/main/resources/sample-data/ledger/*.json` for deterministic offline output. | called from GATHER task | returns `List<LedgerLine>` / `ChartOfAccounts` |
| `ValidateTools` | function-tools class | Implements `checkDebitCreditBalance(entries)` and `validateAccountCodes(entries, coa)`. Pure in-memory checks. | called from VALIDATE task | returns `ValidationResult` |
| `ReportTools` | function-tools class | Implements `formatTrialBalanceSummary(snapshot, entries)` and `writeVarianceCommentary(entries, threshold)`. | called from REPORT task | returns `TrialBalanceSummary` / `String` |
| `ApprovalGate` | workflow-step helper (plain class) | Holds the workflow in a waiting state until `CloseRunEntity.status` transitions to the next active phase (signalled by a `StepApproved` event) or a `StepRejected` event triggers a replay. | polled from `CloseRunWorkflow` approval-gate steps | accept → advance / reject → replay |
| `ReconciliationScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `ClosePackage`, `JournalEntrySet`, `LedgerSnapshot`. Output: `ReconciliationResult{score, rationale}`. | called from `evalStep` | returns score |
| `CloseRunView` | `View` | Read model: one row per close run for the UI. | `CloseRunEntity` events | `CloseRunEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record LedgerLine(
    String accountCode,
    String description,
    BigDecimal debitAmount,
    BigDecimal creditAmount,
    LocalDate postingDate
) {}

record ChartOfAccounts(List<AccountEntry> accounts) {}

record AccountEntry(String code, String name, String type) {}  // type: ASSET, LIABILITY, EQUITY, REVENUE, EXPENSE

record LedgerSnapshot(
    String entity,
    String period,
    List<LedgerLine> lines,
    ChartOfAccounts chartOfAccounts,
    Instant gatheredAt
) {}

record JournalEntry(
    String entryId,
    String debitAccount,
    String creditAccount,
    BigDecimal amount,
    String narration,
    Optional<LocalDate> reversalDate
) {}

record JournalEntrySet(
    List<JournalEntry> entries,
    Instant validatedAt
) {}

record TrialBalanceSummary(
    BigDecimal totalDebits,
    BigDecimal totalCredits,
    BigDecimal variance,
    boolean balanced
) {}

record ClosePackage(
    String title,
    String period,
    String entity,
    TrialBalanceSummary trialBalance,
    List<JournalEntry> journalEntries,
    String varianceCommentary,
    Instant writtenAt
) {}

record ReconciliationResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record ApprovalRecord(
    String step,          // "GATHER" or "VALIDATE"
    String approver,
    String decision,      // "APPROVED" or "REJECTED"
    Optional<String> comment,
    Instant decidedAt
) {}

record CloseRunRecord(
    String closeRunId,
    Optional<String> entity,
    Optional<String> period,
    Optional<LedgerSnapshot> ledgerSnapshot,
    Optional<JournalEntrySet> journalEntrySet,
    Optional<ClosePackage> closePackage,
    Optional<ReconciliationResult> reconciliation,
    List<ApprovalRecord> approvals,
    CloseRunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum CloseRunStatus {
    CREATED, GATHERING, AWAITING_GATHER_APPROVAL,
    VALIDATING, AWAITING_VALIDATE_APPROVAL,
    REPORTING, EVALUATED, FAILED
}
```

Events on `CloseRunEntity`: `CloseRunCreated`, `GatherStarted`, `LedgerDataGathered`, `StepApproved`, `StepRejected`, `ValidateStarted`, `EntriesValidated`, `ReportStarted`, `CloseReportWritten`, `ReconciliationScored`, `CloseRunFailed`.

Every nullable lifecycle field on the `CloseRunRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/close-runs` — body `{ entity, period }` → `{ closeRunId }`.
- `GET /api/close-runs` — list all close runs, newest-first.
- `GET /api/close-runs/{id}` — one close run.
- `POST /api/close-runs/{id}/approve` — body `{ step, approver, comment }` → `204`.
- `POST /api/close-runs/{id}/reject` — body `{ step, approver, comment }` → `204`.
- `GET /api/close-runs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Month-End Closer</title>`.

The App UI tab is a two-column layout: a left rail with the live list of close runs (status pill + entity + period + age) and a right pane with the selected close run's detail — ledger snapshot table, journal entry list, close package sections, reconciliation score chip, and approval/rejection history.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — HITL approval gate (application-level)**: After each of the first two phases completes, `CloseRunWorkflow` parks in an `approvalGate` step. The workflow polls `CloseRunEntity.status` until it sees either a `StepApproved` or `StepRejected` event recorded via `POST /api/close-runs/{id}/approve` or `.../reject`. On approval, the workflow advances to the next phase. On rejection, the workflow replays the step: the agent re-runs the GATHER or VALIDATE task (consuming another iteration budget) and the result is re-presented for approval. The approver identity, decision, and timestamp are recorded on the entity as `StepApproved{step, approver, comment, decidedAt}` or `StepRejected{step, approver, comment, decidedAt}` — every approval decision is durably auditable. The REPORT phase does not have an approval gate; its output is automatically forwarded to `evalStep`.
- **E1 — `on-decision-eval`**: runs immediately after `CloseReportWritten` lands, as `evalStep` inside the workflow. `ReconciliationScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): debits must equal credits across all journal entries (balance check), every journal entry's `debitAccount` and `creditAccount` must appear in the chart of accounts (account-code validity), every accrual entry (`narration` contains "accrual") must carry a non-empty `reversalDate` (accrual reversal completeness), and the `TrialBalanceSummary.variance.abs()` must be within the configured materiality threshold (default `BigDecimal.ZERO` for the sample). Emits `ReconciliationScored{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `CloseAgent` → `prompts/close-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits close run for entity `ACME-US` / period `2026-05`; within 120 s (including two manual approvals) the run reaches `EVALUATED` with non-empty ledger lines, ≥ 2 journal entries, and a reconciliation score chip on the card.
2. **J2** — Accounting rejects the gather phase output; the workflow replays `gatherStep`; the agent reruns the GATHER task; the rerun result is presented for a second approval; the pipeline eventually completes correctly.
3. **J3** — A close run whose mock-LLM trajectory produces a journal entry referencing an account code absent from the chart of accounts is scored 1 with a rationale naming the invalid code; the UI flags the card.
4. **J4** — Each task's instructions and tool calls are logged at `INFO` with the closeRunId tag; the GATHER task's log shows only GATHER-tool calls, the VALIDATE task's log shows only VALIDATE-tool calls, the REPORT task's log shows only REPORT-tool calls. The approval events for each phase are also present in the log.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named month-end-closer demonstrating the sequential-pipeline x finance-analysis
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-finance-analysis-month-end-closer. Java package
io.akka.samples.monthendcloser. Akka 3.6.0. HTTP port 9754.

Components to wire (exactly):

- 1 AutonomousAgent CloseAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/close-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  GATHER, VALIDATE, and REPORT tool sets are ALL registered on the agent; phase order is
  enforced by the HITL approval gate at the workflow level, NOT by conditional .tools(...)
  wiring. HTTP port 9754.

- 1 Workflow CloseRunWorkflow per closeRunId with six steps:
  * gatherStep — emits GatherStarted on the entity, then calls componentClient
    .forAutonomousAgent(CloseAgent.class, "agent-" + closeRunId).runSingleTask(
      TaskDef.instructions("Entity: " + entity + "\nPeriod: " + period + "\nPhase: GATHER\n
      Use fetchLedgerLines and fetchChartOfAccounts to gather the full ledger snapshot.")
        .metadata("closeRunId", closeRunId)
        .metadata("phase", "GATHER")
        .taskType(CloseTasks.GATHER_LEDGER_DATA)
    ). Reads result to get LedgerSnapshot. Writes CloseRunEntity.recordLedgerSnapshot(snapshot).
    WorkflowSettings.stepTimeout 60s.
  * approvalGate(GATHER) — polls CloseRunEntity.status until VALIDATING (StepApproved) or
    until a StepRejected event triggers step replay (back to gatherStep). stepTimeout 3600s
    (1 hour — waits for human). On rejection writes GatherStarted again and loops back to
    gatherStep.
  * validateStep — emits ValidateStarted, then runSingleTask with TaskDef.instructions
    (formatValidateContext(ledgerSnapshot)) and metadata.phase = "VALIDATE", taskType
    VALIDATE_ENTRIES. Writes CloseRunEntity.recordJournalEntrySet(entries). stepTimeout 60s.
  * approvalGate(VALIDATE) — same pattern; waits for StepApproved → REPORTING or StepRejected
    → replay validateStep. stepTimeout 3600s.
  * reportStep — emits ReportStarted, then runSingleTask with TaskDef.instructions
    (formatReportContext(journalEntrySet, ledgerSnapshot, entity, period)) and
    metadata.phase = "REPORT", taskType WRITE_CLOSE_REPORT. Writes
    CloseRunEntity.recordClosePackage(closePackage). stepTimeout 60s.
  * evalStep — runs the deterministic ReconciliationScorer over (closePackage, journalEntrySet,
    ledgerSnapshot) and writes CloseRunEntity.recordReconciliation(result). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(CloseRunWorkflow::error). The error step writes
  CloseRunFailed and ends.

- 1 EventSourcedEntity CloseRunEntity (one per closeRunId). State CloseRunRecord{closeRunId,
  entity: Optional<String>, period: Optional<String>, ledgerSnapshot: Optional<LedgerSnapshot>,
  journalEntrySet: Optional<JournalEntrySet>, closePackage: Optional<ClosePackage>,
  reconciliation: Optional<ReconciliationResult>, approvals: List<ApprovalRecord>,
  status: CloseRunStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  CloseRunStatus enum: CREATED, GATHERING, AWAITING_GATHER_APPROVAL, VALIDATING,
  AWAITING_VALIDATE_APPROVAL, REPORTING, EVALUATED, FAILED.
  Events: CloseRunCreated{entity, period}, GatherStarted, LedgerDataGathered{ledgerSnapshot},
  StepApproved{step, approver, comment, decidedAt},
  StepRejected{step, approver, comment, decidedAt}, ValidateStarted,
  EntriesValidated{journalEntrySet}, ReportStarted, CloseReportWritten{closePackage},
  ReconciliationScored{reconciliation}, CloseRunFailed{reason}.
  Commands: create, startGather, recordLedgerSnapshot, approveStep, rejectStep, startValidate,
  recordJournalEntrySet, startReport, recordClosePackage, recordReconciliation, fail,
  getCloseRun. emptyState() returns CloseRunRecord.initial("") with all Optional fields as
  Optional.empty() and no commandContext() reference (Lesson 3).

- 1 View CloseRunView with row type CloseRunRow that mirrors CloseRunRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes CloseRunEntity events. ONE
  query getAllCloseRuns: SELECT * AS closeRuns FROM close_run_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * CloseRunEndpoint at /api with POST /close-runs (body {entity, period}; mints closeRunId;
    calls CloseRunEntity.create(entity, period); then starts CloseRunWorkflow with id
    "workflow-" + closeRunId; returns {closeRunId}), GET /close-runs (list from
    getAllCloseRuns, sorted newest-first), GET /close-runs/{id} (one row),
    POST /close-runs/{id}/approve (body {step, approver, comment}; calls
    CloseRunEntity.approveStep; returns 204), POST /close-runs/{id}/reject (same for reject),
    GET /close-runs/sse (Server-Sent Events forwarded from the view's stream-updates), and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- CloseTasks.java declaring three Task<R> constants:
    GATHER_LEDGER_DATA = Task.name("Gather ledger data").description("Fetch ledger lines and
      chart of accounts for the given entity and period").resultConformsTo(LedgerSnapshot.class);
    VALIDATE_ENTRIES = Task.name("Validate entries").description("Check debit/credit balance
      and validate account codes for the proposed journal entry set")
      .resultConformsTo(JournalEntrySet.class);
    WRITE_CLOSE_REPORT = Task.name("Write close report").description("Compose a ClosePackage
      with trial balance summary, journal entry list, and variance commentary")
      .resultConformsTo(ClosePackage.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {GATHER, VALIDATE, REPORT}. Each function-tool method is annotated with
  the constant phase (use a custom annotation if the SDK's @FunctionTool does not carry a
  phase field — the registry is built at startup if so).

- GatherTools.java — @FunctionTool fetchLedgerLines(String entity, String period) ->
  List<LedgerLine> reading from src/main/resources/sample-data/ledger/<entity>-<period>.json;
  @FunctionTool fetchChartOfAccounts(String entity) -> ChartOfAccounts reading from the
  matching coa/<entity>.json file.

- ValidateTools.java — @FunctionTool checkDebitCreditBalance(List<JournalEntry>) ->
  ValidationResult (debits.sum == credits.sum); @FunctionTool validateAccountCodes(
  List<JournalEntry>, ChartOfAccounts) -> ValidationResult (every debitAccount and
  creditAccount appears in coa.accounts[].code).

- ReportTools.java — @FunctionTool formatTrialBalanceSummary(LedgerSnapshot,
  JournalEntrySet) -> TrialBalanceSummary (totalDebits, totalCredits, variance, balanced);
  @FunctionTool writeVarianceCommentary(JournalEntrySet, BigDecimal threshold) -> String
  (one paragraph naming entries whose amounts exceed threshold).

- ApprovalGate.java — workflow-step helper that reads CloseRunEntity.status and blocks the
  step until either a StepApproved or StepRejected event is recorded. On StepApproved the
  helper returns an advance signal; on StepRejected it returns a replay signal. The workflow
  step transitions accordingly.

- ReconciliationScorer.java — pure deterministic logic (no LLM). Inputs: ClosePackage,
  JournalEntrySet, LedgerSnapshot. Outputs: ReconciliationResult with score and rationale.
  Four checks, one point per check satisfied, starting from a base of 1: debit/credit balance
  (totalDebits == totalCredits in TrialBalanceSummary), account-code validity (every entry's
  debitAccount and creditAccount in the chart of accounts), accrual reversal completeness
  (every accrual entry has a non-empty reversalDate), and variance within threshold
  (trialBalance.variance.abs() <= materialityThreshold). Score range 1-5. Rationale names
  the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9754 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-data/ledger/*.json — three files keyed by entity+period pairs
  (ACME-US-2026-05, ACME-EU-2026-05, WIDGET-US-2026-04), each carrying 8-12 LedgerLine
  entries with deterministic amounts so GatherTools returns the same snapshot across restarts.

- src/main/resources/sample-data/coa/*.json — one ChartOfAccounts file per entity code,
  listing 15-20 AccountEntry records covering ASSET, LIABILITY, EQUITY, REVENUE, EXPENSE types.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (H1, E1) matching the mechanisms in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root pre-filled for the finance-analysis domain.

- prompts/close-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Month-End Closer", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of close run cards; right = selected-run detail with entity+period header, ledger
  snapshot table, journal entry list, close package summary, reconciliation score chip,
  approval/rejection history). Browser title exactly:
  <title>Akka Sample: Month-End Closer</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(closeRunId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    gather-ledger-data.json — 4 LedgerSnapshot entries, each with 8-12 LedgerLine items for
      ACME-US-2026-05. Each entry's tool_calls array contains 2 calls: fetchLedgerLines +
      fetchChartOfAccounts. Plus 1 deliberately phase-violating entry whose tool_calls array
      starts with checkDebitCreditBalance (a VALIDATE-phase tool called during GATHER) — the
      guardrail rejects it; the mock then falls through to a normal gather sequence.
    validate-entries.json — 4 JournalEntrySet entries each with 3-6 JournalEntry items,
      tool_calls containing checkDebitCreditBalance + validateAccountCodes.
    write-close-report.json — 4 ClosePackage entries, tool_calls containing
      formatTrialBalanceSummary + writeVarianceCommentary. Plus 1 entry with an invalid
      account code in one JournalEntry — ReconciliationScorer scores it 1; J3 verifies this.
- A MockModelProvider.seedFor(closeRunId) helper makes per-run selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CloseAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion CloseTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (gatherStep
  60s, validateStep 60s, reportStep 60s, evalStep 5s, approvalGate(GATHER) 3600s,
  approvalGate(VALIDATE) 3600s, error 5s).
- Lesson 6: every nullable lifecycle field on CloseRunRecord is Optional<T>.
- Lesson 7: CloseTasks.java with GATHER_LEDGER_DATA, VALIDATE_ENTRIES, WRITE_CLOSE_REPORT
  is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9754 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, complex, Akka SDK in narrative, marketing
  tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (CloseAgent). The
  on-decision eval is rule-based (ReconciliationScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: the workflow's HITL approval gate enforces the phase
  order between GATHER and VALIDATE, and between VALIDATE and REPORT. The agent is stateless
  across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
