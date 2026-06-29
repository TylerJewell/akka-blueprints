# SPEC — finance-close-reconciler

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Finance Close Reconciler.
**One-line pitch:** An accountant submits a period-end trial balance and a list of reconciliation rules; one AI agent reads the balance (passed as a task attachment, never as inline prompt text) and returns a structured `ReconciliationReport` — a variance summary plus a per-account-pair finding indicating whether each pair is IN_BALANCE, VARIANCE, or UNMATCHED.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the finance-analysis domain. One `ReconciliationAgent` (AutonomousAgent) carries the entire reconciliation decision; the surrounding components prepare its input, guard its tool calls, collect human sign-off, and attest the audit trail. Four governance mechanisms are wired around the agent:

- A **financial-data sanitizer** runs inside a Consumer between the raw trial-balance submission and the agent call — stripping internal account codes, confidential margin percentages, and entity-specific identifiers so sensitive commercial data does not appear in LLM call logs.
- A **before-tool-call guardrail** intercepts every GL-write tool call proposed by the agent: validates the account code exists in the chart of accounts, that the debit/credit pair is balanced, and that the period has not already been closed. A rejected proposal forces the agent to revise its tool call within the same task.
- A **human-in-the-loop** step holds the workflow after a `ReconciliationReport` lands; the accountant reviews the variance list and either approves it (advancing to attestation) or rejects it with a note (returning the workflow to a revised-run state).
- A **CI attestation gate** asserts, at build time, that every reconciled period in the test fixtures has a complete event chain (`PeriodSubmitted → BalanceSanitized → ReconciliationStarted → ReportRecorded → SignOffGranted → AttestationCompleted`) before the artifact is promoted.

The blueprint shows that the single-agent pattern does not mean "ungoverned" — four independent checks sit on either side of the one decision-making LLM call, addressing the audit and SOX/IFRS requirements specific to financial close.

## 3. User-facing flows

The user opens the App UI tab.

1. The accountant picks a **period** from a dropdown (Q1 / Q2 / Q3 / Q4 of the current fiscal year) or enters a custom period label.
2. The accountant pastes a trial balance in CSV format into the **Trial Balance** textarea (or picks one of three seeded examples — a 12-account IFRS operating entity, a 20-account US GAAP manufacturing entity, a 6-account inter-company eliminations set).
3. The accountant picks a **reconciliation rule set** from a dropdown (IFRS-operating, GAAP-manufacturing, interco-eliminations) or pastes a custom list of account-pair rules.
4. The accountant clicks **Submit for reconciliation**. The UI POSTs to `/api/periods` and receives a `periodId`.
5. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SANITIZED` — the masked balance is visible in the card detail, with a note of which fields were masked.
6. Within ~10–30 s, the workflow's `reconcileStep` completes. The card transitions to `RECONCILING` then `REPORT_RECORDED`. The report appears: a top-level status badge (IN_BALANCE / HAS_VARIANCES / INCOMPLETE), a variance summary paragraph, and a per-account-pair table (account pair, expected balance, actual balance, variance amount, finding).
7. The card moves to `AWAITING_SIGNOFF`. A prominent **Approve** / **Reject** button pair appears in the detail pane. The accountant reads the variance list and clicks **Approve**.
8. Within ~1 s the workflow advances to `ATTESTED`. The card shows an attestation chip with the approver id and timestamp.
9. The accountant can submit another period; the live list keeps all history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ReconciliationEndpoint` | `HttpEndpoint` | `/api/periods/*` — submit, list, get, SSE, sign-off; serves `/api/metadata/*`. | — | `PeriodEntity`, `ReconciliationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PeriodEntity` | `EventSourcedEntity` | Per-period lifecycle: submitted → sanitized → reconciling → report recorded → awaiting sign-off → attested. Source of truth. | `ReconciliationEndpoint`, `BalanceSanitizer`, `ReconciliationWorkflow` | `ReconciliationView` |
| `BalanceSanitizer` | `Consumer` | Subscribes to `PeriodSubmitted` events; masks confidential fields; calls `PeriodEntity.attachSanitized`. | `PeriodEntity` events | `PeriodEntity` |
| `ReconciliationWorkflow` | `Workflow` | One workflow per period. Steps: `awaitSanitizedStep` → `reconcileStep` → `awaitSignOffStep` → `attestStep`. | started by `BalanceSanitizer` once sanitized event lands | `ReconciliationAgent`, `PeriodEntity` |
| `ReconciliationAgent` | `AutonomousAgent` | The one decision-making LLM. Receives reconciliation rules in the task definition and the sanitized balance as a task attachment; proposes GL writes via tool calls validated by `WriteGuardrail`; returns `ReconciliationReport`. | invoked by `ReconciliationWorkflow` | returns report |
| `WriteGuardrail` | supporting class | Before-tool-call hook registered on `ReconciliationAgent`. Validates every proposed GL write. | called by agent loop | pass/reject |
| `AttestationScorer` | supporting class | Deterministic completeness scorer run in `attestStep`. Verifies the event chain is complete and all findings are covered. | called by `ReconciliationWorkflow` | `AttestationResult` |
| `ReconciliationView` | `View` | Read model: one row per period for the UI. | `PeriodEntity` events | `ReconciliationEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record AccountRule(
    String ruleId,
    String accountPair,       // e.g. "1000/2000"
    String description,
    String toleranceAmount,   // e.g. "0.01" (absolute) or "0.5%" (relative)
    boolean material
) {}

record TrialBalanceSubmission(
    String periodId,
    String periodLabel,       // e.g. "Q2 FY2026"
    String rawBalance,        // CSV text
    List<AccountRule> rules,
    String submittedBy,
    Instant submittedAt
) {}

record SanitizedBalance(
    String maskedBalance,           // CSV with confidential fields masked
    List<String> maskedFieldTypes   // e.g. ["margin-pct","entity-code","internal-account"]
) {}

record AccountFinding(
    String ruleId,
    String accountPair,
    BalanceFinding finding,
    BigDecimal expectedBalance,
    BigDecimal actualBalance,
    BigDecimal variance,
    String note
) {}
enum BalanceFinding { IN_BALANCE, VARIANCE, UNMATCHED }

record ReconciliationReport(
    ReportStatus status,
    String summary,
    List<AccountFinding> findings,
    List<String> proposedGlWrites,  // tool-call outputs that passed the guardrail
    Instant reconciledAt
) {}
enum ReportStatus { IN_BALANCE, HAS_VARIANCES, INCOMPLETE }

record SignOff(
    String approvedBy,
    SignOffDecision decision,
    String note,
    Instant decidedAt
) {}
enum SignOffDecision { APPROVED, REJECTED }

record AttestationResult(
    boolean complete,
    String attestedBy,
    Instant attestedAt
) {}

record Period(
    String periodId,
    Optional<TrialBalanceSubmission> submission,
    Optional<SanitizedBalance> sanitized,
    Optional<ReconciliationReport> report,
    Optional<SignOff> signOff,
    Optional<AttestationResult> attestation,
    PeriodStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum PeriodStatus {
    SUBMITTED, SANITIZED, RECONCILING, REPORT_RECORDED,
    AWAITING_SIGNOFF, ATTESTED, REJECTED, FAILED
}
```

Events on `PeriodEntity`: `PeriodSubmitted`, `BalanceSanitized`, `ReconciliationStarted`, `ReportRecorded`, `SignOffRequested`, `SignOffGranted`, `SignOffRejected`, `AttestationCompleted`, `PeriodFailed`.

Every nullable lifecycle field on the `Period` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/periods` — body `{ periodLabel, rawBalance, rules: [AccountRule], submittedBy }` → `{ periodId }`.
- `GET /api/periods` — list all periods, newest-first.
- `GET /api/periods/{id}` — one period.
- `GET /api/periods/sse` — Server-Sent Events; one event per state transition.
- `POST /api/periods/{id}/signoff` — body `{ approvedBy, decision: APPROVED|REJECTED, note }` → `204`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Finance Close Reconciler</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted periods (status pill + report badge + age) and a right pane with the selected period's detail — submitted rules list, masked balance preview, report summary, account-finding table, sign-off controls, and attestation chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every tool call proposed by `ReconciliationAgent`. Asserts the proposed account code exists in the submitted chart of accounts, the debit and credit sides are equal, the period has not been previously closed, and the write amount does not exceed a configurable materiality threshold. On failure, returns a structured `invalid-tool-call` rejection to the agent loop so the task retries within its iteration budget.
- **H1 — human-in-the-loop sign-off** (`application` flavor): after `ReportRecorded` lands, the workflow pauses in `awaitSignOffStep` indefinitely. The accountant calls `POST /api/periods/{id}/signoff` with `APPROVED` or `REJECTED`. Approval advances to `attestStep`; rejection writes `SignOffRejected` and transitions to `REJECTED`.
- **S1 — financial-data sanitizer** (`sector` flavor): runs inside `BalanceSanitizer` Consumer between the raw submission and the agent invocation. Strips confidential margin percentages, internal entity codes, and account identifiers not needed by the reconciliation rules. The raw balance is preserved on the entity for audit but is never the input seen by the model.
- **CI1 — attestation gate** (`attestation-gate` flavor): a build-time check asserts that every period in the test fixtures has a complete event chain before the artifact is promoted. Implemented as a unit-test assertion in `AttestationGateTest.java` that fails the build if any period fixture is missing a terminal event.

## 9. Agent prompts

- `ReconciliationAgent` → `prompts/reconciliation-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached sanitized trial balance, walk every account-pair rule, and return one `AccountFinding` per rule. Proposed GL writes are emitted as structured tool calls, each of which is validated by `WriteGuardrail` before execution.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Accountant submits the IFRS-operating seed balance; within 30 s the report appears with one finding per submitted rule, an IN_BALANCE or HAS_VARIANCES badge, and a sign-off prompt.
2. **J2** — The agent proposes a GL write with a non-existent account code (mock path); `WriteGuardrail` rejects it; the second iteration produces a valid write proposal; the UI never displays the blocked proposal.
3. **J3** — The accountant approves the report; the workflow advances to `ATTESTED`; the attestation chip shows the approver and timestamp.
4. **J4** — A balance containing internal entity codes like `ENTITY_MARGIN_PCT=0.32` is submitted; the LLM call log shows only `[MASKED-MARGIN-PCT]`; the entity's `submission.rawBalance` retains the raw text for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named finance-close-reconciler demonstrating the single-agent × finance-analysis
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-finance-analysis-finance-close-reconciler. Java package
io.akka.samples.financereconciliationagent. Akka 3.6.0. HTTP port 9365.

Components to wire (exactly):

- 1 AutonomousAgent ReconciliationAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/reconciliation-agent.md>) and
  .capability(TaskAcceptance.of(RECONCILE_PERIOD).maxIterationsPerTask(4)). The task receives
  reconciliation rules as its instruction text and the sanitized balance as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: ReconciliationReport{status: ReportStatus, summary: String,
  findings: List<AccountFinding>, proposedGlWrites: List<String>, reconciledAt: Instant}.
  The agent is configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the agent
  loop retries the tool call within its 4-iteration budget.

- 1 Workflow ReconciliationWorkflow per periodId with four steps:
  * awaitSanitizedStep — polls PeriodEntity.getPeriod every 1s; on period.sanitized().isPresent()
    advances to reconcileStep. WorkflowSettings.stepTimeout 15s.
  * reconcileStep — emits ReconciliationStarted, then calls componentClient.forAutonomousAgent(
    ReconciliationAgent.class, "reconciler-" + periodId).runSingleTask(
      TaskDef.instructions(formatRules(period.submission.rules))
        .attachment("balance.csv", period.sanitized.maskedBalance.getBytes())
    ) — returns a taskId, then forTask(taskId).result(RECONCILE_PERIOD) to fetch the report.
    On success calls PeriodEntity.recordReport(report), then emits SignOffRequested.
    WorkflowSettings.stepTimeout 90s with defaultStepRecovery maxRetries(2).failoverTo(
    ReconciliationWorkflow::error).
  * awaitSignOffStep — polls PeriodEntity.getPeriod every 2s; on
    period.signOff().isPresent() checks decision. If APPROVED, advances to attestStep.
    If REJECTED, calls PeriodEntity.fail("sign-off-rejected") and ends.
    WorkflowSettings.stepTimeout 86400s (24h — human latency).
  * attestStep — runs a deterministic AttestationScorer (NOT an LLM call) over the full period
    event chain: verifies all required events are present and ordered, all findings are
    covered, and the sign-off is APPROVED. Emits AttestationCompleted{complete: true/false,
    attestedBy: signOff.approvedBy, attestedAt: Instant.now()}.
    WorkflowSettings.stepTimeout 5s. error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity PeriodEntity (one per periodId). State Period{periodId: String,
  submission: Optional<TrialBalanceSubmission>, sanitized: Optional<SanitizedBalance>,
  report: Optional<ReconciliationReport>, signOff: Optional<SignOff>,
  attestation: Optional<AttestationResult>, status: PeriodStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. PeriodStatus enum: SUBMITTED, SANITIZED,
  RECONCILING, REPORT_RECORDED, AWAITING_SIGNOFF, ATTESTED, REJECTED, FAILED.
  Events: PeriodSubmitted{submission}, BalanceSanitized{sanitized}, ReconciliationStarted{},
  ReportRecorded{report}, SignOffRequested{}, SignOffGranted{signOff}, SignOffRejected{signOff},
  AttestationCompleted{attestation}, PeriodFailed{reason}.
  Commands: submit, attachSanitized, markReconciling, recordReport, requestSignOff,
  grantSignOff, rejectSignOff, attest, fail, getPeriod.
  emptyState() returns Period.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 Consumer BalanceSanitizer subscribed to PeriodEntity events; on PeriodSubmitted runs a
  masking pipeline over rawBalance: strips columns matching patterns for margin-pct
  (e.g., "%_MARGIN_PCT"), entity codes (e.g., "ENTITY_CODE_*"), and internal account
  identifiers not present in the submitted rules list. Computes the list of masked field
  types, builds SanitizedBalance, then calls PeriodEntity.attachSanitized(sanitized). After
  attachSanitized lands, the same Consumer starts a ReconciliationWorkflow with id =
  "recon-" + periodId.

- 1 View ReconciliationView with row type PeriodRow (mirrors Period minus
  submission.rawBalance — the audit log keeps the raw; the view holds the sanitized form for
  the UI). Table updater consumes PeriodEntity events. ONE query getAllPeriods:
  SELECT * AS periods FROM period_view. No WHERE status filter — Akka cannot auto-index enum
  columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ReconciliationEndpoint at /api with:
    POST /periods (body {periodLabel, rawBalance, rules: [{ruleId, accountPair, description,
      toleranceAmount, material}], submittedBy}; mints periodId; calls PeriodEntity.submit;
      returns {periodId}),
    GET /periods (list from getAllPeriods, sorted newest-first),
    GET /periods/{id} (one row),
    GET /periods/sse (Server-Sent Events forwarded from the view's stream-updates),
    POST /periods/{id}/signoff (body {approvedBy, decision, note}; calls
      PeriodEntity.grantSignOff or PeriodEntity.rejectSignOff based on decision; returns 204),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ReconciliationTasks.java declaring one Task<R> constant: RECONCILE_PERIOD = Task.name(
  "Reconcile period").description("Read the attached trial balance and produce a
  ReconciliationReport with one AccountFinding per submitted rule")
  .resultConformsTo(ReconciliationReport.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records AccountRule, TrialBalanceSubmission, SanitizedBalance, AccountFinding,
  BalanceFinding, ReconciliationReport, ReportStatus, SignOff, SignOffDecision,
  AttestationResult, Period, PeriodStatus.

- WriteGuardrail.java implementing the before-tool-call hook. Reads the proposed GL write
  from the agent tool call, runs the four checks listed in eval-matrix.yaml G1, and either
  passes the tool call through or returns Guardrail.reject(<structured-error>) to force the
  agent loop to retry with a revised tool call.

- AttestationScorer.java — pure deterministic logic (no LLM). Inputs: full event list for a
  Period and the submitted AccountRules. Outputs: AttestationResult. Scoring logic documented
  in Javadoc on the class. Verifies event chain completeness and that every ruleId in the
  submission has a corresponding AccountFinding in the report.

- AttestationGateTest.java — a JUnit test that loads every fixture in
  src/test/resources/fixtures/periods/ and asserts AttestationScorer.score(period).complete
  == true. This test is wired into the Maven verify phase; a failing test breaks the build
  and the artifact is not promoted.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9365 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. ReconciliationAgent.definition() binds
  the configured provider via the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/account-rules.jsonl with 3 seeded rule sets:
  an 8-rule IFRS-operating checklist (cash, AR, AP, prepaid, accruals, tax, equity, retained
  earnings), a 12-rule GAAP-manufacturing checklist (adds inventory, WIP, fixed assets,
  depreciation), and a 4-rule interco-eliminations checklist.

- src/main/resources/sample-events/seed-balances.jsonl with 3 paired example trial balances:
  a synthetic IFRS operating entity (14 accounts, ~1500 characters CSV), a synthetic GAAP
  manufacturing entity (22 accounts, ~2000 characters CSV), and a synthetic inter-company
  set (8 accounts, ~600 characters CSV). Each contains 2–3 confidential margin fields so
  S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 4 controls (G1, H1, S1, CI1) matching the
  mechanisms in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.financial-records = true,
  decisions.authority_level = recommend-only (the agent's report is advisory — the accountant
  approves), oversight.human_in_loop = true, failure.failure_modes including
  "variance-missed", "invalid-gl-write", "confidential-data-leakage",
  "incomplete-event-chain"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/reconciliation-agent.md loaded as the agent system prompt.

- README.md at the project root.

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of period cards; right = selected-period detail with submitted rules, masked
  balance preview, report summary, finding table, sign-off controls, attestation chip).
  Browser title exactly: <title>Akka Sample: Finance Close Reconciler</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(periodId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    reconcile-period.json — 8 ReconciliationReport entries covering all three ReportStatus
      values. Each entry has a summary paragraph and a `findings` array with one
      AccountFinding per rule in the matched rule set. Each AccountFinding has a non-empty
      accountPair, a BigDecimal expectedBalance, actualBalance, variance, and a note. Plus
      2 deliberately INVALID tool-call entries (one proposing a write to a non-existent
      account code; one with an unbalanced debit/credit pair) — the guardrail blocks both,
      exercising the retry path. The mock selects an invalid entry on the FIRST iteration
      of every 3rd period (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(periodId) helper makes per-period selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ReconciliationAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ReconciliationTasks.java
  MUST exist.
- Lesson 4: every workflow step that calls an agent or waits on a human has an explicit
  stepTimeout (reconcileStep 90s, awaitSanitizedStep 15s, awaitSignOffStep 86400s,
  attestStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Period row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: ReconciliationTasks.java with RECONCILE_PERIOD = Task.name(...).description(...)
  .resultConformsTo(ReconciliationReport.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9365 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  and themeVariables block. Without these, state names render black-on-black.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ReconciliationAgent).
  The attestation scorer is rule-based (AttestationScorer.java) and does NOT make an LLM
  call.
- The trial balance is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated reconcileStep uses TaskDef.attachment(...).
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism.
  It intercepts tool calls BEFORE they execute, not after. This is the correct cut-point for
  blocking invalid GL writes before they alter any state.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
