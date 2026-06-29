# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## ResearchBrief (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `briefId` | `String` | no | UUID, also the workflow id |
| `topic` | `String` | no | The submitted research topic |
| `status` | `BriefStatus` | no | Lifecycle state |
| `findings` | `Optional<FindingsBundle>` | yes | Researcher output; null until `FindingsAttached` |
| `analysis` | `Optional<AnalyticalReport>` | yes | Analyst output; null until `AnalysisAttached` |
| `synthesised` | `Optional<SynthesisedBrief>` | yes | Merged brief; null until `BriefSynthesised` |
| `failureReason` | `Optional<String>` | yes | Set on `BriefBlocked` or `BriefDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `EvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Brief creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `ResearchView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record TopicRequest(String topic, String requestedBy) {}
record WorkPlan(String researchQuery, String analyticalQuestion) {}
record Finding(String headline, String source, String content) {}
record FindingsBundle(List<Finding> findings, Instant gatheredAt) {}
record AnalyticalReport(String thesis, List<String> implications, Instant analysedAt) {}
record SynthesisedBrief(String summary, FindingsBundle findings, AnalyticalReport analysis,
                        String guardrailVerdict, Instant synthesisedAt) {}
```

## Status enum

```java
enum BriefStatus { PLANNING, IN_PROGRESS, SYNTHESISED, DEGRADED, BLOCKED }
```

## Events

### ResearchBriefEntity

| Event | Trigger |
|---|---|
| `BriefCreated` | Workflow creates the brief (`createBrief`) |
| `FindingsAttached` | Researcher returns a `FindingsBundle` |
| `AnalysisAttached` | Analyst returns an `AnalyticalReport` |
| `BriefSynthesised` | Coordinator synthesis passes the guardrail |
| `BriefDegraded` | A worker timed out; synthesised from partial input |
| `BriefBlocked` | Guardrail rejected the synthesised brief |
| `EvalScored` | `EvalSampler` recorded a 1–5 score |

### RequestQueue

| Event | Trigger |
|---|---|
| `TopicSubmitted` | `enqueueTopic(topic, requestedBy)` from endpoint or simulator |

Fields: `{ briefId, topic, requestedBy, submittedAt }`.
