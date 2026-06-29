# Data Model — Financial Model Builder

---

## Records

### LineItem

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `name` | String | No | Accounting line name (e.g. `Revenue`, `CostOfGoodsSold`) |
| `period` | String | No | Reporting period (e.g. `2025-Q4`) |
| `value` | BigDecimal | No | Numeric value |
| `unit` | String | No | Unit of measure (e.g. `USD`, `shares`) |
| `sourceRef` | String | No | Stable reference into the source filing; used for provenance tracing |

### FilingData

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `filingId` | String | No | Unique identifier for the parsed filing |
| `ticker` | String | No | Exchange ticker symbol |
| `period` | String | No | Reporting period |
| `items` | List\<LineItem\> | No | All line items extracted from the filing (may be empty) |
| `extractedAt` | Instant | No | Timestamp of extraction |

### ModelRow

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `rowId` | String | No | Unique row identifier within the model |
| `label` | String | No | Human-readable row label (e.g. `Gross Margin`) |
| `period` | String | No | Period this row applies to |
| `value` | BigDecimal | No | Computed value |
| `derivation` | String | No | `LineItem.sourceRef` this row was derived from; required for provenance |

### Assumption

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `key` | String | No | Machine key for the assumption (e.g. `revenue-growth-rate`) |
| `description` | String | No | Human-readable description |
| `assumedValue` | BigDecimal | No | Numeric value assumed; scored against line-item median in E1 |

### FinancialModel

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `modelId` | String | No | Unique model identifier |
| `ticker` | String | No | Ticker this model covers |
| `period` | String | No | Period this model covers |
| `rows` | List\<ModelRow\> | No | All model rows |
| `assumptions` | List\<Assumption\> | No | All assumptions |
| `builtAt` | Instant | No | Timestamp of model construction |

### ValidationFlag

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `flagId` | String | No | Unique flag identifier |
| `severity` | String | No | `CRITICAL`, `WARNING`, or `INFO` |
| `description` | String | No | Human-readable description of the issue |
| `affectedRow` | String | No | `ModelRow.rowId` affected, or `ASSUMPTION` |

### ValidationReport

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `flags` | List\<ValidationFlag\> | No | All flags raised during validation |
| `confidence` | int | No | 0–100 confidence score; `< 60` triggers a failing E1 check |
| `validatedAt` | Instant | No | Timestamp of validation |

### EvalResult

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `score` | int | No | 1–5 filing fidelity score |
| `rationale` | String | No | Which of the four checks passed and which failed |
| `evaluatedAt` | Instant | No | Timestamp of evaluation |

### ModelRecord (aggregate state)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `modelId` | String | No | Unique model identifier |
| `ticker` | Optional\<String\> | Yes | Set after `ModelCreated` |
| `filingData` | Optional\<FilingData\> | Yes | Set after `FilingExtracted` |
| `model` | Optional\<FinancialModel\> | Yes | Set after `ModelBuilt` |
| `validation` | Optional\<ValidationReport\> | Yes | Set after `ModelValidated` |
| `analystNote` | Optional\<String\> | Yes | Set after `ModelApproved` or `ModelRejected` |
| `eval` | Optional\<EvalResult\> | Yes | Set after `EvaluationScored` |
| `status` | ModelStatus | No | Current pipeline status |
| `createdAt` | Instant | No | Entity creation timestamp |
| `finishedAt` | Optional\<Instant\> | Yes | Set when terminal state reached |

---

## Enums

### ModelStatus

```java
enum ModelStatus {
    CREATED,         // entity created, pipeline not yet started
    EXTRACTING,      // extractStep running
    EXTRACTED,       // FilingData ready
    BUILDING,        // buildStep running
    BUILT,           // FinancialModel ready
    VALIDATING,      // validateStep running
    VALIDATED,       // ValidationReport ready
    PENDING_REVIEW,  // waiting for analyst decision
    APPROVED,        // analyst approved; eval pending
    REJECTED,        // analyst rejected; terminal
    EVALUATED,       // EvaluationScored received; terminal
    FAILED           // unrecoverable pipeline error; terminal
}
```

---

## Events

| Event | Payload fields | State transition |
|---|---|---|
| `ModelCreated` | `modelId`, `ticker`, `period`, `createdAt` | → CREATED |
| `ExtractStarted` | `modelId` | CREATED → EXTRACTING |
| `FilingExtracted` | `modelId`, `filingData` | EXTRACTING → EXTRACTED |
| `BuildStarted` | `modelId` | EXTRACTED → BUILDING |
| `ModelBuilt` | `modelId`, `model` | BUILDING → BUILT |
| `ValidateStarted` | `modelId` | BUILT → VALIDATING |
| `ModelValidated` | `modelId`, `validation` | VALIDATING → VALIDATED |
| `ReviewRequested` | `modelId` | VALIDATED → PENDING_REVIEW |
| `ModelApproved` | `modelId`, `analystNote` | PENDING_REVIEW → APPROVED |
| `ModelRejected` | `modelId`, `analystNote`, `finishedAt` | PENDING_REVIEW → REJECTED |
| `EvaluationScored` | `modelId`, `evalResult`, `finishedAt` | APPROVED → EVALUATED |
| `GuardrailRejected` | `modelId`, `rejectedTool`, `currentPhase`, `toolPhase` | audit-only; no transition |
| `ModelFailed` | `modelId`, `reason`, `finishedAt` | any active phase → FAILED |

---

## View row

`FinancialModelView` projects one row per `modelId`:

| Column | Type | Source event |
|---|---|---|
| `modelId` | String | `ModelCreated` |
| `ticker` | String | `ModelCreated` |
| `period` | String | `ModelCreated` |
| `status` | ModelStatus | updated by every state-changing event |
| `evalScore` | Optional\<Integer\> | `EvaluationScored` |
| `hasGuardrailRejection` | boolean | `GuardrailRejected` |
| `createdAt` | Instant | `ModelCreated` |
| `updatedAt` | Instant | last event timestamp |

---

## Task definitions

```java
public final class ModelTasks {
    public static final String EXTRACT_FINANCIALS = "EXTRACT_FINANCIALS";
    public static final String BUILD_MODEL        = "BUILD_MODEL";
    public static final String VALIDATE_MODEL     = "VALIDATE_MODEL";
}
```

---

## Phase-tagged tools

| Tool method | Class | Phase tag |
|---|---|---|
| `parseFiling(ticker, period)` | `ExtractTools` | EXTRACT |
| `fetchLineItem(sourceRef)` | `ExtractTools` | EXTRACT |
| `computeRatio(items, formula)` | `BuildTools` | BUILD |
| `projectFigure(items, assumption)` | `BuildTools` | BUILD |
| `crossCheckAssumption(assumption, items)` | `ValidateTools` | VALIDATE |
| `detectAnomalies(rows, items)` | `ValidateTools` | VALIDATE |

`ModelPhaseGuardrail` reads the phase tag from each tool's annotation and compares it to the phase carried in the agent task context. A mismatch causes a before-tool-call rejection.
