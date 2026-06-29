# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently -- the wire value is the raw value or `null`.

## Report (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `reportId` | `String` | no | UUID, also the workflow id |
| `topic` | `String` | no | The submitted writing topic |
| `status` | `ReportStatus` | no | Lifecycle state |
| `sources` | `Optional<SourceBundle>` | yes | SourceResearcher output; null until `SourcesAttached` |
| `draft` | `Optional<DraftSection>` | yes | DraftWriter output; null until `DraftAttached` |
| `assembled` | `Optional<AssembledReport>` | yes | Merged report; null until `ReportAssembled` |
| `failureReason` | `Optional<String>` | yes | Set on `ReportBlocked` or `ReportDegraded` |
| `qualityScore` | `Optional<Integer>` | yes | 1-5; null until `ReportScored` |
| `qualityRationale` | `Optional<String>` | yes | One-line quality justification |
| `createdAt` | `Instant` | no | Report creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `ReportView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record TopicRequest(String topic, String requestedBy) {}
record WritingPlan(String sourceQuery, String narrativeBrief) {}
record Source(String title, String reference, String excerpt) {}
record SourceBundle(List<Source> sources, Instant gatheredAt) {}
record DraftSection(String narrativeText, List<String> keyPoints, Instant draftedAt) {}
record AssembledReport(String headline, String body, SourceBundle sourceList,
                       String guardrailVerdict, Instant assembledAt) {}
```

## Status enum

```java
enum ReportStatus { PLANNING, IN_PROGRESS, ASSEMBLED, DEGRADED, BLOCKED }
```

## Events

### ReportEntity

| Event | Trigger |
|---|---|
| `ReportCreated` | Workflow creates the report (`createReport`) |
| `SourcesAttached` | SourceResearcher returns a `SourceBundle` |
| `DraftAttached` | DraftWriter returns a `DraftSection` |
| `ReportAssembled` | Coordinator assembly passes the guardrail |
| `ReportDegraded` | A worker timed out; assembled from partial input |
| `ReportBlocked` | Guardrail rejected the assembled report |
| `ReportScored` | `QualitySampler` recorded a 1-5 quality score |

### TopicQueue

| Event | Trigger |
|---|---|
| `TopicSubmitted` | `enqueueTopic(topic, requestedBy)` from endpoint or simulator |

Fields: `{ reportId, topic, requestedBy, submittedAt }`.
