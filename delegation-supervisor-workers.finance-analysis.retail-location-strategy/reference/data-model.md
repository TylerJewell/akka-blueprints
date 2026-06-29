# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## CandidateSite (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `siteId` | `String` | no | UUID, also the workflow id |
| `address` | `String` | no | Street address of the candidate site |
| `city` | `String` | no | City of the candidate site |
| `region` | `String` | no | State or region code |
| `status` | `SiteStatus` | no | Lifecycle state |
| `marketAssessment` | `Optional<MarketAssessment>` | yes | Market Analyst output; null until `MarketAssessmentAttached` |
| `demographicAssessment` | `Optional<DemographicAssessment>` | yes | Demographics Analyst output; null until `DemographicAssessmentAttached` |
| `recommendation` | `Optional<SiteRecommendation>` | yes | Synthesised recommendation; null until `SiteRecommended` or `SiteNotRecommended` |
| `failureReason` | `Optional<String>` | yes | Set on `SiteBlocked` or `SiteDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `EvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Site evaluation creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `LocationView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record SiteSubmission(String address, String city, String region, String requestedBy) {}

record ScoringPlan(String marketQuery, String demographicQuestion) {}

record MarketAssessment(
    double trafficScore,
    double competitorDensity,
    String tradeAreaClassification,
    List<String> keyInsights,
    Instant assessedAt
) {}

record DemographicAssessment(
    double populationScore,
    double incomeAlignmentScore,
    String consumerProfile,
    List<String> keyInsights,
    Instant assessedAt
) {}

record SiteRecommendation(
    String summary,
    MarketAssessment marketAssessment,
    DemographicAssessment demographicAssessment,
    double score,
    LocationVerdict verdict,
    String guardrailVerdict,
    Instant synthesisedAt
) {}
```

## Status enums

```java
enum SiteStatus { SCORING, IN_PROGRESS, RECOMMENDED, NOT_RECOMMENDED, DEGRADED, BLOCKED }
enum LocationVerdict { RECOMMENDED, NOT_RECOMMENDED }
```

## Events

### CandidateSiteEntity

| Event | Trigger |
|---|---|
| `SiteCreated` | Workflow creates the evaluation (`createSite`) |
| `MarketAssessmentAttached` | Market Analyst returns a `MarketAssessment` |
| `DemographicAssessmentAttached` | Demographics Analyst returns a `DemographicAssessment` |
| `SiteRecommended` | Coordinator synthesis passes the guardrail with verdict=RECOMMENDED |
| `SiteNotRecommended` | Coordinator synthesis passes the guardrail with verdict=NOT_RECOMMENDED |
| `SiteDegraded` | An analyst timed out; synthesised from partial input |
| `SiteBlocked` | Guardrail rejected the synthesised recommendation |
| `EvalScored` | `EvalSampler` recorded a 1–5 quality score |

### SiteQueue

| Event | Trigger |
|---|---|
| `SiteSubmitted` | `enqueueSite(address, city, region, requestedBy)` from endpoint or simulator |

Fields: `{ siteId, address, city, region, requestedBy, submittedAt }`.
