# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## AdvisorySession (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `sessionId` | `String` | no | UUID, also the workflow id |
| `description` | `String` | no | The submitted startup brief description |
| `status` | `SessionStatus` | no | Lifecycle state |
| `marketSnapshot` | `Optional<MarketSnapshot>` | yes | MarketResearcher output; null until `MarketSnapshotAttached` |
| `channelPlan` | `Optional<ChannelPlan>` | yes | GrowthAnalyst output; null until `ChannelPlanAttached` |
| `guidance` | `Optional<GtmGuidance>` | yes | Merged GTM guidance; null until `GuidanceSynthesised` |
| `failureReason` | `Optional<String>` | yes | Set on `SessionDegraded` or guardrail rejection |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `GuidanceEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Session creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `AdvisoryView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record StartupBrief(String description, String submittedBy) {}

record BriefDecomposition(String marketResearchQuery, String channelStrategyQuestion) {}

record BuyerSegment(String name, String painPoint) {}

record MarketSnapshot(
    String tam,
    List<String> competitors,
    List<BuyerSegment> buyerSegments,
    Instant researchedAt
) {}

record ChannelRecommendation(String channel, String rationale, String effort) {}

record ChannelPlan(
    List<ChannelRecommendation> channels,
    String primaryChannel,
    Instant analysedAt
) {}

record GtmGuidance(
    String summary,
    MarketSnapshot marketSnapshot,
    ChannelPlan channelPlan,
    String guardrailVerdict,
    Instant synthesisedAt
) {}
```

## Status enum

```java
enum SessionStatus { SCOPING, IN_PROGRESS, ADVISED, DEGRADED }
```

## Events

### AdvisorySessionEntity

| Event | Trigger |
|---|---|
| `SessionCreated` | Workflow creates the session (`createSession`) |
| `MarketSnapshotAttached` | MarketResearcher returns a `MarketSnapshot` |
| `ChannelPlanAttached` | GrowthAnalyst returns a `ChannelPlan` |
| `GuidanceSynthesised` | AdvisorCoordinator synthesis completes successfully |
| `SessionDegraded` | A worker timed out or a tool call was blocked; synthesised from partial input |
| `GuidanceEvalScored` | `EvalSampler` recorded a 1–5 score |

### BriefQueue

| Event | Trigger |
|---|---|
| `BriefSubmitted` | `enqueueBrief(description, submittedBy)` from endpoint or simulator |

Fields: `{ sessionId, description, submittedBy, submittedAt }`.
