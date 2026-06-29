# Financial Model Builder — Specification

## 1. System overview

**System name:** Financial Model Builder

A user submits a ticker symbol and reporting period. One `FinancialModelAgent` walks SEC filings through three task phases — EXTRACT line items from source filings, BUILD a financial model from those line items, VALIDATE the model's assumptions against source data — with each phase gated on prior phase output. A mandatory analyst review gate follows before the model is scored by `FilingFidelityScorer`.

The pipeline is fully event-sourced. Every phase transition, guardrail event, analyst decision, and eval result is persisted as a domain event on `FinancialModelEntity`.

---

## 2. What this demonstrates

**Coordination pattern:** sequential-pipeline
**Domain:** finance-analysis

| Governance mechanism | Component | Hook |
|---|---|---|
| Before-tool-call guardrail | `ModelPhaseGuardrail` | Blocks tool calls that belong to a later phase |
| HITL review gate | `FinancialModelPipelineWorkflow` + `FinancialModelEntity` | Workflow pauses at `reviewStep`; analyst approves or rejects |
| On-decision eval | `FilingFidelityScorer` | Runs after approval; scores filing fidelity |

---

## 3. User-facing flows

1. User submits `{ ticker, period }` to `POST /api/models`.
2. Response returns `{ modelId }` immediately.
3. Status progresses: `CREATED → EXTRACTING → EXTRACTED → BUILDING → BUILT → VALIDATING → VALIDATED → PENDING_REVIEW`.
4. Analyst opens the App UI, sees the model in PENDING_REVIEW, reviews phase panels, and clicks Approve or Reject.
5. On approval: status moves to `APPROVED → EVALUATED` with a score chip (1–5).
6. On rejection: status moves to `REJECTED` (terminal).

Seed tickers for local development (mock filing data included): `AAPL`, `MSFT`, `GOOGL`.

---

## 4. Components

| Component | Kind | Package path |
|---|---|---|
| `FinancialModelAgent` | AutonomousAgent | `agent` |
| `FinancialModelPipelineWorkflow` | Workflow | `workflow` |
| `FinancialModelEntity` | EventSourcedEntity | `entity` |
| `ExtractTools` | Tool class | `tools` |
| `BuildTools` | Tool class | `tools` |
| `ValidateTools` | Tool class | `tools` |
| `ModelPhaseGuardrail` | Guardrail | `guardrail` |
| `FilingFidelityScorer` | Scorer | `scorer` |
| `FinancialModelView` | View | `view` |
| `ModelEndpoint` | HttpEndpoint | `endpoint` |
| `AppEndpoint` | HttpEndpoint | `endpoint` |
| `ModelTasks` | Constants class | `model` |

---

## 5. Data model

```java
// Domain records
record LineItem(String name, String period, BigDecimal value, String unit, String sourceRef) {}
record FilingData(String filingId, String ticker, String period, List<LineItem> items, Instant extractedAt) {}

record ModelRow(String rowId, String label, String period, BigDecimal value, String derivation) {}
record Assumption(String key, String description, BigDecimal assumedValue) {}
record FinancialModel(String modelId, String ticker, String period, List<ModelRow> rows,
                      List<Assumption> assumptions, Instant builtAt) {}

record ValidationFlag(String flagId, String severity, String description, String affectedRow) {}
record ValidationReport(List<ValidationFlag> flags, int confidence, Instant validatedAt) {}

record EvalResult(int score, String rationale, Instant evaluatedAt) {}

// Aggregate state
record ModelRecord(
    String modelId,
    Optional<String> ticker,
    Optional<FilingData> filingData,
    Optional<FinancialModel> model,
    Optional<ValidationReport> validation,
    Optional<String> analystNote,
    Optional<EvalResult> eval,
    ModelStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ModelStatus {
    CREATED, EXTRACTING, EXTRACTED, BUILDING, BUILT, VALIDATING, VALIDATED,
    PENDING_REVIEW, APPROVED, REJECTED, EVALUATED, FAILED
}
```

**Events:** `ModelCreated`, `ExtractStarted`, `FilingExtracted`, `BuildStarted`, `ModelBuilt`,
`ValidateStarted`, `ModelValidated`, `ReviewRequested`, `ModelApproved`, `ModelRejected`,
`EvaluationScored`, `GuardrailRejected`, `ModelFailed`

**Task constants (`ModelTasks.java`):**

```java
public final class ModelTasks {
    public static final String EXTRACT_FINANCIALS = "EXTRACT_FINANCIALS";
    public static final String BUILD_MODEL        = "BUILD_MODEL";
    public static final String VALIDATE_MODEL     = "VALIDATE_MODEL";
}
```

---

## 6. API contract

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/api/models` | `{ ticker, period }` | `{ modelId }` |
| GET | `/api/models` | — | `List<ModelRecord>` |
| GET | `/api/models/{id}` | — | `ModelRecord` |
| POST | `/api/models/{id}/approve` | `{ analystNote }` | 204 |
| POST | `/api/models/{id}/reject` | `{ analystNote }` | 204 |
| GET | `/api/models/sse` | — | SSE stream |
| GET | `/api/metadata/readme` | — | README.md |
| GET | `/api/metadata/risk-survey` | — | risk-survey.yaml |
| GET | `/api/metadata/eval-matrix` | — | eval-matrix.yaml |
| GET | `/` | — | redirect → `/app/index.html` |
| GET | `/app/*` | — | static assets |

Full payload examples: `reference/api-contract.md`.

---

## 7. UI

Browser title: `<title>Akka Sample: Financial Model Builder</title>`

Five-tab structure using attribute-based tab switching (`data-tab` / `data-panel` — Lesson 26):

| Tab | Label | Content |
|---|---|---|
| 1 | Overview | Eyebrow + headline, try-it card, how-it-works card, components card, API card |
| 2 | Architecture | Stat tiles, four mermaid diagrams from PLAN.md, component table |
| 3 | Risk Survey | 7 sub-tabs rendering risk-survey.yaml |
| 4 | Eval Matrix | 5-column table for H1 and E1 |
| 5 | App UI | Two-column live UI: model list left, detail panel right |

**App UI detail panel — phase panels:**

- **EXTRACT panel** — table of extracted `LineItem` rows (name, period, value, unit, sourceRef).
- **BUILD panel** — `ModelRow` table + `Assumption` list.
- **VALIDATE panel** — `ValidationFlag` list with severity badges (CRITICAL=red, WARNING=yellow, INFO=blue).
- **REVIEW panel** — shown when status is `PENDING_REVIEW`; Approve button, Reject button, analystNote text input.
- **EVAL panel** — score chip (1–5) + rationale text.

Status pill colours: `PENDING_REVIEW`=orange, `APPROVED`=green, `REJECTED`=red, `EVALUATED`=teal, `FAILED`=red. Red dot on model card if a `GuardrailRejected` event exists for that model.

Apply Lesson 24 CSS token overrides for the dark-themed mermaid diagrams. See `reference/ui-mockup.md` for full markup structure.

---

## 8. Governance

### H1 — HITL analyst approval gate

`ModelValidated` transitions `FinancialModelEntity` to `PENDING_REVIEW` and emits `ReviewRequested`. `FinancialModelPipelineWorkflow` pauses at `reviewStep` by waiting on the entity state.

The analyst submits one of:
- `POST /api/models/{id}/approve` — entity emits `ModelApproved`, status → `APPROVED`.
- `POST /api/models/{id}/reject` — entity emits `ModelRejected`, status → `REJECTED` (terminal).

The workflow only advances to `evalStep` after observing `APPROVED` status. Rejection bypasses eval entirely.

### E1 — Filing fidelity eval (on-decision-eval)

`FilingFidelityScorer` runs inside `evalStep` after `ModelApproved`. Four deterministic checks (one point each):

1. **Row provenance** — every `ModelRow.derivation` traces to a `LineItem.sourceRef`.
2. **Assumption deviation** — no `Assumption.assumedValue` deviates more than 20% from the median of matching `LineItem` values.
3. **Confidence threshold** — `ValidationReport.confidence >= 60`.
4. **No critical flags** — `ValidationReport.flags` contains no entry with `severity = "CRITICAL"`.

Score range: 1–5. Entity emits `EvaluationScored`; status → `EVALUATED`.

---

## 9. Agent prompts

`FinancialModelAgent` loads its system prompt from:

```
prompts/financial-model-agent.md
```

---

## 10. Acceptance journeys

Four journeys defined in `reference/user-journeys.md`:

| ID | Name |
|---|---|
| J1 | Submit ticker and get an approved model |
| J2 | Phase-gate guardrail blocks misordered tool call |
| J3 | Analyst rejects a model |
| J4 | FilingFidelityScorer flags aggressive assumption |

---

## 11. Implementation directives

```
MAVEN:
  groupId:    io.akka.samples
  artifactId: sequential-pipeline-finance-analysis-model-builder
  version:    1.0-SNAPSHOT
  akkaVersion: 3.6.0
  javaPackage: io.akka.samples.financialmodelbuilder
  port:       9286

COMPONENTS:
  FinancialModelAgent         → agent/FinancialModelAgent.java          (AutonomousAgent)
  FinancialModelPipelineWorkflow → workflow/FinancialModelPipelineWorkflow.java  (Workflow)
  FinancialModelEntity        → entity/FinancialModelEntity.java         (EventSourcedEntity)
  ExtractTools                → tools/ExtractTools.java
  BuildTools                  → tools/BuildTools.java
  ValidateTools               → tools/ValidateTools.java
  ModelPhaseGuardrail         → guardrail/ModelPhaseGuardrail.java       (before-tool-call)
  FilingFidelityScorer        → scorer/FilingFidelityScorer.java         (on-decision-eval)
  FinancialModelView          → view/FinancialModelView.java             (View)
  ModelEndpoint               → endpoint/ModelEndpoint.java
  AppEndpoint                 → endpoint/AppEndpoint.java
  ModelTasks                  → model/ModelTasks.java

KEY FILES:
  src/main/resources/application.conf       (port 9286, dev-mode)
  src/main/resources/static-resources/      (compiled Angular assets)
  prompts/financial-model-agent.md

AGENT STEPS:
  Workflow steps: extractStep → buildStep → validateStep → reviewStep → evalStep
  Each step calls FinancialModelAgent with the appropriate ModelTasks constant.
  reviewStep pauses on FinancialModelEntity state; does not call the agent.

GUARDRAIL:
  ModelPhaseGuardrail intercepts before-tool-call.
  Reads current entity phase from context.
  Rejects tool calls belonging to a phase other than the current one.
  Emits GuardrailRejected event on rejection (audit-only; no state change).

SCORER:
  FilingFidelityScorer is deterministic (no LLM call).
  Runs in evalStep after ModelApproved.
  Implements the four checks in §8 E1.
  Returns score 1–5 via EvaluationScored event.

UI:
  Five-tab structure: data-tab / data-panel attribute switching (Lesson 26).
  Lesson 24 CSS overrides for mermaid theme variables.
  App UI tab SSE stream from GET /api/models/sse (Lesson 25 — no polling).
  Status pill colours per §7.

CONSTRAINTS:
  Lesson 1:  One entity per aggregate; FinancialModelEntity is the sole state owner.
  Lesson 4:  Workflow drives the pipeline; entity only handles commands and emits events.
  Lesson 6:  GuardrailRejected is audit-only; never transition entity state on guardrail block.
  Lesson 7:  All agent outputs are typed records (FilingData, FinancialModel, ValidationReport).
  Lesson 8:  Tool classes are plain Java; no Akka dependencies inside ExtractTools/BuildTools/ValidateTools.
  Lesson 9:  FilingFidelityScorer is deterministic; no LLM call inside the scorer.
  Lesson 10: reviewStep waits on entity state, does not poll — use Workflow.asyncEffect to observe.
  Lesson 11: HITL path: ModelApproved/ModelRejected are entity commands, not workflow-internal signals.
  Lesson 12: EvalResult is written to entity via EvaluationScored before workflow ends.
  Lesson 13: ModelFailed is terminal; workflow does not retry after FAILED.
  Lesson 23: Mock filing data for AAPL, MSFT, GOOGL must be present for local tests without a live API.
  Lesson 24: Mermaid theme block in every diagram.
  Lesson 25: App UI tab uses EventSource on /api/models/sse; never setTimeout polling.
  Lesson 26: Tab switching via data-tab/data-panel attributes; no JavaScript class toggling.
```

---

## 12. Post-scaffolding workflow

After `/akka:specify @SPEC.md` completes, the following chain runs automatically:

```
/akka:plan   → generates PLAN.md
/akka:tasks  → generates tasks.md
/akka:implement → writes all Java sources, tests, and Angular assets
/akka:build  → compiles, runs tests, starts the service locally
```

Verify by submitting `POST /api/models` with `{ "ticker": "AAPL", "period": "2025-Q4" }` and following the status to `PENDING_REVIEW`, then approving via the App UI.

---

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
