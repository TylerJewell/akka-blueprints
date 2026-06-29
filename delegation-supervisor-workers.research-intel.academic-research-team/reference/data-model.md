# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## ResearchDigest (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `digestId` | `String` | no | UUID, also the workflow id |
| `query` | `String` | no | The submitted research query |
| `status` | `DigestStatus` | no | Lifecycle state |
| `publications` | `Optional<PublicationBundle>` | yes | PaperScout output; null until `PublicationsAttached` |
| `trends` | `Optional<TrendReport>` | yes | DomainAnalyst output; null until `TrendsAttached` |
| `synthesised` | `Optional<SynthesisedDigest>` | yes | Merged digest; null until `DigestSynthesised` |
| `failureReason` | `Optional<String>` | yes | Set on `DigestBlocked` or `DigestDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `DigestEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Digest creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `DigestView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record QueryRequest(String query, String requestedBy) {}
record ScoutingPlan(String scoutingDirective, String interpretationQuestion) {}
record Publication(String title, String doi, String venue, String abstract_) {}
record PublicationBundle(List<Publication> publications, Instant scoutedAt) {}
record TrendReport(String thesis, List<String> emergingAreas, List<String> gaps, Instant analysedAt) {}
record SynthesisedDigest(String summary, PublicationBundle publications, TrendReport trends,
                         String guardrailVerdict, Instant synthesisedAt) {}
```

## Status enum

```java
enum DigestStatus { QUEUED, SCANNING, SYNTHESISED, DEGRADED, BLOCKED }
```

## Events

### ResearchDigestEntity

| Event | Trigger |
|---|---|
| `DigestCreated` | Workflow creates the digest (`createDigest`) |
| `PublicationsAttached` | PaperScout returns a `PublicationBundle` |
| `TrendsAttached` | DomainAnalyst returns a `TrendReport` |
| `DigestSynthesised` | Coordinator synthesis passes the citation guardrail |
| `DigestDegraded` | A worker timed out; synthesised from partial input |
| `DigestBlocked` | Citation guardrail rejected the synthesised digest |
| `DigestEvalScored` | `EvalSampler` recorded a 1–5 score |

### QueryQueue

| Event | Trigger |
|---|---|
| `QuerySubmitted` | `enqueueQuery(query, requestedBy)` from endpoint or simulator |

Fields: `{ digestId, query, requestedBy, submittedAt }`.
