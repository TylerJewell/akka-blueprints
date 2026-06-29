# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## AnalysisJob (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `jobId` | `String` | no | UUID, also the workflow id |
| `question` | `String` | no | The submitted natural-language financial question |
| `status` | `JobStatus` | no | Lifecycle state |
| `sqlPlan` | `Optional<SqlPlan>` | yes | Coordinator's translated SQL; null until `SqlPlanAttached` |
| `dataSet` | `Optional<DataSet>` | yes | DataRetriever output; null until `DataSetAttached` |
| `modelResult` | `Optional<ModelResult>` | yes | StatisticsModeler output; null until `ModelResultAttached` |
| `report` | `Optional<DataReport>` | yes | Synthesised report; null until `JobCompleted` |
| `failureReason` | `Optional<String>` | yes | Set on `JobBlocked` or `JobDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5 NL2SQL fidelity score; null until `TranslationEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Job creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `AnalysisView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record NlQuestion(String question, String requestedBy) {}

record SqlPlan(String queryId, String sql, String targetTable, String safetyVerdict) {}

record DataRow(List<String> columns, List<List<String>> rows, Instant queriedAt) {}
record DataSet(String queryId, DataRow data, int rowCount, Instant retrievedAt) {}

record ModelMetric(String metricName, double value, String interpretation) {}
record ModelResult(String modelType, List<ModelMetric> metrics, Instant modelledAt) {}

record DataReport(String summary, DataSet dataSet, ModelResult modelResult,
                  Instant synthesisedAt) {}
```

## Status enum

```java
enum JobStatus { TRANSLATING, RUNNING, COMPLETE, DEGRADED, BLOCKED }
```

## Events

### AnalysisJobEntity

| Event | Trigger |
|---|---|
| `JobCreated` | Workflow creates the job (`createJob`) |
| `SqlPlanAttached` | Coordinator returns a `SqlPlan` and the guardrail accepts it |
| `DataSetAttached` | DataRetriever returns a `DataSet` |
| `ModelResultAttached` | StatisticsModeler returns a `ModelResult` |
| `JobCompleted` | Coordinator synthesis succeeds; both workers returned |
| `JobDegraded` | A worker timed out; synthesised from partial input |
| `JobBlocked` | Guardrail detected a destructive SQL keyword |
| `TranslationEvalScored` | `EvalSampler` recorded a 1–5 NL2SQL fidelity score |

### QueryQueue

| Event | Trigger |
|---|---|
| `QuestionSubmitted` | `enqueueQuestion(question, requestedBy)` from endpoint or simulator |

Fields: `{ jobId, question, requestedBy, submittedAt }`.

## Modelled BigQuery schema (in-process)

The DataRetriever operates against these in-process tables; no live database connection is required.

| Table | Columns |
|---|---|
| `revenue` | `date`, `product_line`, `region`, `amount_usd` |
| `expenses` | `date`, `category`, `department`, `amount_usd` |
| `accounts` | `account_id`, `account_name`, `balance_usd`, `currency`, `last_activity` |
| `trades` | `trade_id`, `symbol`, `quantity`, `price_usd`, `direction`, `executed_at` |
| `positions` | `symbol`, `quantity`, `avg_cost_usd`, `current_price_usd`, `unrealised_pnl`, `as_of` |
