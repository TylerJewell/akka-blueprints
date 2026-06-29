# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## AnalysisReport (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `reportId` | `String` | no | UUID, also the workflow id |
| `question` | `String` | no | The submitted market question |
| `status` | `ReportStatus` | no | Lifecycle state |
| `indicators` | `Optional<IndicatorBundle>` | yes | DataCollector output; null until `IndicatorsAttached` |
| `interpretation` | `Optional<EconomicInterpretation>` | yes | Economist output; null until `InterpretationAttached` |
| `synthesised` | `Optional<SynthesisedReport>` | yes | Merged report; null until `ReportPublished` |
| `failureReason` | `Optional<String>` | yes | Set on `ReportBlocked` or `ReportDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `ReportEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Report creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `AnalysisView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record MarketQuestion(String question, String requestedBy) {}
record ResearchPlan(String dataQuery, String interpretiveQuestion) {}
record Indicator(String name, String value, String unit, String source) {}
record IndicatorBundle(List<Indicator> indicators, Instant collectedAt) {}
record EconomicInterpretation(String thesis, List<String> implications, Instant interpretedAt) {}
record SynthesisedReport(String summary, IndicatorBundle indicators,
                         EconomicInterpretation interpretation,
                         String disclaimerVerdict, Instant synthesisedAt) {}
```

## Status enum

```java
enum ReportStatus { FRAMING, IN_PROGRESS, PUBLISHED, DEGRADED, BLOCKED }
```

## Events

### AnalysisReportEntity

| Event | Trigger |
|---|---|
| `ReportCreated` | Workflow creates the report (`createReport`) |
| `IndicatorsAttached` | DataCollector returns an `IndicatorBundle` |
| `InterpretationAttached` | Economist returns an `EconomicInterpretation` |
| `ReportPublished` | Coordinator synthesis passes the guardrail |
| `ReportDegraded` | A worker timed out; synthesised from partial input |
| `ReportBlocked` | Guardrail rejected the synthesised report |
| `ReportEvalScored` | `EvalSampler` recorded a 1–5 score |

### RequestQueue

| Event | Trigger |
|---|---|
| `QuestionSubmitted` | `enqueueQuestion(question, requestedBy)` from endpoint or simulator |

Fields: `{ reportId, question, requestedBy, submittedAt }`.
