# SPEC — fraud-flagging-team

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–12 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Fraud Flagging Team.
**One-line pitch:** An analyst submits (or a feed drips) a customer transaction; a supervisor agent delegates the transaction to fraud-scoring, compliance, and risk-assessment workers, synthesizes their findings into a fraud verdict, and pauses for a human analyst to confirm before any account action runs.

## 2. What this blueprint demonstrates

The delegation-supervisor-workers coordination pattern in the finance-analysis domain: one supervisor plans the work, three specialized workers run in parallel on the same input, and the supervisor synthesizes their typed outputs into a single decision. The governance pattern is a human gate on an irreversible action — the verdict is advisory until a human analyst confirms it, a guardrail blocks the account-action tool until confirmation, transaction data is redacted before it reaches logs, and every verdict is evaluated for false-positive and false-negative signal.

## 3. User-facing flows

1. An analyst submits a transaction (`customerId`, `amount`, `transactionRef`, `memo`) on the App UI tab, or the background feed drips one. A case opens in `ANALYZING`.
2. The workflow fans the transaction out to the three workers and waits for all three typed results.
3. The supervisor agent synthesizes the worker outputs into a verdict. The case moves to `FLAGGED` (fraud suspected) or `CLEARED` (no action needed; terminal).
4. A `FLAGGED` case shows in the review list with the supervisor rationale and the three worker findings, plus Confirm and Dismiss buttons.
5. The analyst clicks Confirm. A before-tool-call guardrail verifies the case is confirmed, then the gated account action runs and the case reaches `ACTIONED`.
6. The analyst clicks Dismiss with a note. The case reaches `DISMISSED`; no account action runs.
7. A `FLAGGED` case left untouched past the review window auto-escalates to `ESCALATED`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| FraudReviewWorkflow | Workflow | Orchestrates delegation, synthesis, analyst gate, gated action | TransactionConsumer | FraudScoringWorker, ComplianceWorker, RiskAssessmentWorker, SupervisorAgent, CaseEntity |
| SupervisorAgent | AutonomousAgent | Synthesizes worker outputs into a fraud verdict | FraudReviewWorkflow | CaseEntity (via workflow) |
| FraudScoringWorker | Agent | Scores the transaction for fraud risk, returns FraudScore | FraudReviewWorkflow | — |
| ComplianceWorker | Agent | Checks the transaction against compliance rules, returns ComplianceFinding | FraudReviewWorkflow | — |
| RiskAssessmentWorker | Agent | Assesses per-customer risk tier, returns CustomerRiskAssessment | FraudReviewWorkflow | — |
| CaseEntity | EventSourcedEntity | Holds one fraud case lifecycle | FraudReviewWorkflow, FraudEndpoint | CasesView |
| TransactionQueue | EventSourcedEntity | Inbound transaction queue | TransactionSimulator, FraudEndpoint | TransactionConsumer |
| CasesView | View | CQRS read model of cases | CaseEntity | FraudEndpoint, StuckCaseMonitor |
| TransactionConsumer | Consumer | Opens a case + starts a workflow per queued transaction | TransactionQueue | FraudReviewWorkflow |
| TransactionSimulator | TimedAction | Drips canned transactions every 30s | — | TransactionQueue |
| StuckCaseMonitor | TimedAction | Escalates flagged cases past the review window | CasesView | CaseEntity |
| FraudEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI, simulator | CaseEntity, TransactionQueue, CasesView |
| AppEndpoint | HttpEndpoint | Serves the static UI | browser | static-resources |

## 5. Data model

Authoritative — `/akka:implement` writes records exactly as specified. Full field tables in `reference/data-model.md`.

`FraudCase` (CaseEntity state and CasesView row): `String id`, `String customerId`, `Optional<String> transactionRef`, `Optional<Double> amount`, `CaseStatus status`, `Optional<Instant> openedAt`, `Optional<Double> fraudScore`, `Optional<String> fraudSignals`, `Optional<String> complianceNotes`, `Optional<Boolean> compliant`, `Optional<String> riskTier`, `Optional<String> riskRationale`, `Optional<String> supervisorRationale`, `Optional<Double> verdictConfidence`, `Optional<Instant> flaggedAt`, `Optional<Instant> clearedAt`, `Optional<Instant> confirmedAt`, `Optional<String> confirmedBy`, `Optional<String> analystNote`, `Optional<Instant> dismissedAt`, `Optional<Instant> escalatedAt`, `Optional<Instant> actionedAt`, `Optional<String> actionTaken`.

`CaseStatus` enum: `ANALYZING`, `FLAGGED`, `CLEARED`, `CONFIRMED`, `ACTIONED`, `DISMISSED`, `ESCALATED`.

Events: `CaseOpened`, `AnalysisRecorded`, `CaseFlagged`, `CaseCleared`, `CaseConfirmed`, `CaseDismissed`, `CaseEscalated`, `CaseActioned`. TransactionQueue event: `TransactionQueued`.

Worker typed outputs: `FraudScore(double score, String signals)`, `ComplianceFinding(boolean compliant, String notes)`, `CustomerRiskAssessment(String tier, String rationale)`, `SupervisorVerdict(boolean flag, double confidence, String rationale)`.

## 6. API contract

Top-level surface (payload schemas in `reference/api-contract.md`):

```
POST /api/cases                       -> { caseId }
POST /api/cases/{caseId}/confirm      -> 200 | 404
POST /api/cases/{caseId}/dismiss      -> 200 | 404
GET  /api/cases ?status=...           -> { cases: [FraudCase, ...] }
GET  /api/cases/{caseId}              -> FraudCase
GET  /api/cases/sse                   -> Server-Sent Events of FraudCase
GET  /api/metadata/eval-matrix        -> text/yaml
GET  /api/metadata/risk-survey        -> text/yaml
GET  /api/metadata/readme             -> text/markdown
GET  /                                -> 302 /app/index.html
GET  /app/{*path}                     -> static-resources/{*path}
```

## 7. UI

A single self-contained `src/main/resources/static-resources/index.html` — no `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Browser title: `<title>Akka Sample: Fraud Flagging Team</title>`.

Five tabs — see `reference/ui-mockup.md`: Overview, Architecture, Risk Survey, Eval Matrix, App UI. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index, and removed panels are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the four mermaid diagrams with the Akka theme variables plus the state-label and edge-label CSS overrides (Lesson 24). The App UI tab submits a transaction, streams the case list via SSE, and shows Confirm/Dismiss buttons on `FLAGGED` cases.

## 8. Governance

Controls in `eval-matrix.yaml`; deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **hitl · application (H1):** FraudReviewWorkflow pauses in `awaitAnalyst` while the case is `FLAGGED`; `POST /api/cases/{id}/confirm` and `.../dismiss` resume it.
- **sanitizer · sector (S1):** a redaction helper masks card numbers, account numbers, and customer identifiers before any transaction field is written to a log line.
- **eval-event · on-decision-eval (E1):** a consumer on `CaseFlagged`/`CaseCleared` records a false-positive/false-negative signal and a verdict-quality score surfaced beside the case.
- **guardrail · before-tool-call (G1):** a before-tool-call guardrail on the account-action step blocks the freeze/notify tool unless `CaseEntity.status == CONFIRMED`.

## 9. Agent prompts

- `prompts/supervisor-agent.md` — synthesizes the three worker outputs into a `SupervisorVerdict`.
- `prompts/fraud-scoring-worker.md` — returns a `FraudScore` from transaction signals.
- `prompts/compliance-worker.md` — returns a `ComplianceFinding` against compliance rules.
- `prompts/risk-assessment-worker.md` — returns a `CustomerRiskAssessment` tier for the customer.

## 10. Acceptance

Inline journeys (full set in `reference/user-journeys.md`):

1. **Submit a transaction and watch it analyze.** POST `/api/cases`; the case appears in `ANALYZING`, then reaches `FLAGGED` or `CLEARED` within ~30s with the three worker findings and the supervisor rationale populated.
2. **Confirm a flagged case.** Confirm a `FLAGGED` case; the gated account action runs and the case reaches `ACTIONED` with a non-null `actionTaken`.
3. **Dismiss a flagged case.** Dismiss with a note; the case reaches `DISMISSED` and no account action runs.
4. **Auto-escalation.** A `FLAGGED` case untouched past the review window reaches `ESCALATED`.

---

## 11. Implementation directives

```
Create a sample named fraud-flagging-team demonstrating the
delegation-supervisor-workers x finance-analysis cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
fraud-flagging-team. Java package io.akka.samples.fraudflaggingteam. Akka 3.6.0.
HTTP port 9778.

Components to wire (exactly):
- 1 AutonomousAgent SupervisorAgent: takes the three worker outputs plus the
  transaction summary and returns a typed SupervisorVerdict{flag, confidence,
  rationale}. definition() declares capability(TaskAcceptance.of(SYNTHESIZE)
  .maxIterationsPerTask(3)). Companion FraudTasks.java declares the SYNTHESIZE
  Task<SupervisorVerdict>. Never silently downgrade to Agent.
- 3 Agents (request/response): FraudScoringWorker.score(tx) -> FraudScore;
  ComplianceWorker.check(tx) -> ComplianceFinding; RiskAssessmentWorker
  .assess(tx) -> CustomerRiskAssessment. Each uses
  effects().systemMessage(prompt).userMessage(tx).responseAs(R.class).thenReply().
- 1 Workflow FraudReviewWorkflow with steps: delegateStep (calls all three
  worker Agents via componentClient.forAgent(); collect FraudScore,
  ComplianceFinding, CustomerRiskAssessment), synthesizeStep (calls
  forAutonomousAgent(SupervisorAgent.class,...).runSingleTask(...) then
  forTask(taskId).result(...); writes AnalysisRecorded then CaseFlagged or
  CaseCleared on CaseEntity), awaitAnalystStep (polls CaseEntity.getCase; on
  FLAGGED self-schedules a 5-second resume timer; on CONFIRMED transitions to
  actionStep; on DISMISSED or ESCALATED ends), actionStep (before-tool-call
  guardrail checks status==CONFIRMED, runs the in-process account-action tool,
  writes CaseActioned). Override settings() with stepTimeout(60s) on
  delegateStep, synthesizeStep, and actionStep, and a defaultStepRecovery with
  maxRetries(2).failoverTo(error).
- 1 EventSourcedEntity CaseEntity holding the FraudCase record with id,
  customerId, and all Optional lifecycle fields per Section 5. Events:
  CaseOpened, AnalysisRecorded, CaseFlagged, CaseCleared, CaseConfirmed,
  CaseDismissed, CaseEscalated, CaseActioned. Commands: openCase, recordAnalysis,
  flag, clear, confirm, dismiss, markEscalated, recordAction, getCase.
  emptyState() returns FraudCase.initial("", "") with no commandContext()
  reference.
- 1 EventSourcedEntity TransactionQueue with a single command
  enqueueTransaction(tx) emitting TransactionQueued.
- 1 View CasesView with row type FraudCase, table updater consuming CaseEntity
  events. ONE query: getAllCases SELECT * AS cases FROM cases_view. No WHERE
  status filter (Akka cannot auto-index enum columns) — filter client-side in
  callers.
- 1 Consumer TransactionConsumer subscribed to TransactionQueue events; on each
  event opens a CaseEntity (fresh UUID) and starts a FraudReviewWorkflow.
- 2 TimedActions: TransactionSimulator (every 30s, reads next line from
  src/main/resources/sample-events/transactions.jsonl and calls
  TransactionQueue.enqueueTransaction); StuckCaseMonitor (every 30s, queries
  CasesView.getAllCases, filters FLAGGED with flaggedAt older than the review
  window, calls CaseEntity.markEscalated).
- 2 HttpEndpoints: FraudEndpoint at /api with cases create, confirm, dismiss,
  cases list (filter client-side from getAllCases), single case, SSE stream,
  and three /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*. ACL open to internet (local-dev only).

Companion files:
- FraudTasks.java declaring the SYNTHESIZE Task<SupervisorVerdict>.
- Records: TransactionInput(String customerId, double amount, String
  transactionRef, String memo), FraudScore(double score, String signals),
  ComplianceFinding(boolean compliant, String notes),
  CustomerRiskAssessment(String tier, String rationale),
  SupervisorVerdict(boolean flag, double confidence, String rationale).
- A Redaction helper that masks card/account/customer identifiers in any
  transaction field before it is logged (control S1).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9778 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/transactions.jsonl with 8 canned
  transaction lines (mix of clearly-clean and suspicious amounts/memos).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 4 controls (H1, S1, E1, G1)
  and matching simplified_view list. regulation_anchors empty.
- risk-survey.yaml at the project root, pre-filling sector (financial services
  / fraud detection), decisions, data.types, capability.*, model.*,
  subjects.children; marking deployer fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root per Section 12 of the authoring guide.
- src/main/resources/static-resources/index.html — single self-contained HTML
  with five tabs (Overview, Architecture, Risk Survey, Eval Matrix, App UI).
  Match the governance.html visual style (dark / yellow accent / Instrument
  Sans / dot-grid). Include the Lesson 24 mermaid CSS overrides and theme
  variables; switch tabs by data-tab/data-panel attribute (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (SupervisorAgent -> SupervisorVerdict, FraudScoringWorker ->
  FraudScore, ComplianceWorker -> ComplianceFinding, RiskAssessmentWorker ->
  CustomerRiskAssessment; see src/main/resources/mock-responses/
  {supervisor-agent,fraud-scoring-worker,compliance-worker,
  risk-assessment-worker}.json with 4-6 entries each). Sets model-provider =
  mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes captured key
  material.

Mock LLM provider — per-agent mock-response shapes when option (a) is chosen:
- fraud-scoring-worker.json: 4-6 FraudScore objects {score: 0.0-1.0, signals:
  short text}.
- compliance-worker.json: 4-6 ComplianceFinding objects {compliant: bool,
  notes: short text}.
- risk-assessment-worker.json: 4-6 CustomerRiskAssessment objects {tier:
  LOW|MEDIUM|HIGH, rationale: short text}.
- supervisor-agent.json: 4-6 SupervisorVerdict objects {flag: bool, confidence:
  0.0-1.0, rationale: short text}.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent (SupervisorAgent) never silently downgraded to Agent.
- Lesson 4: every agent-calling workflow step sets an explicit stepTimeout (60s).
- Lesson 6: Optional<T> for every nullable lifecycle field on the FraudCase row.
- Lesson 7: AutonomousAgent requires the FraudTasks.java companion.
- Lesson 8: verify model names current before locking application.conf.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port 9778 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column, no horizontal scroll.
- Lesson 13: descriptive integration label "Runs out of the box", never T1/T2/T3/T4.
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: mermaid state-label colour + edge-label foreignObject overflow:visible
  + transitionLabelColor #cccccc in index.html.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute; delete dead panels.
- emptyState() never calls commandContext().
- View has no WHERE-on-enum query; filter client-side.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and the PLAN.md.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
