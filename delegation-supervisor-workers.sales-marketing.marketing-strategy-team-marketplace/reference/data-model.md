# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## CampaignBrief (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `briefId` | `String` | no | UUID, also the workflow id |
| `objective` | `String` | no | The submitted campaign objective |
| `status` | `BriefStatus` | no | Lifecycle state |
| `marketInsights` | `Optional<MarketInsightsBundle>` | yes | MarketResearcher output; null until `MarketInsightsAttached` |
| `audienceProfile` | `Optional<AudienceProfile>` | yes | AudienceTargeter output; null until `AudienceProfileAttached` |
| `messagingGuide` | `Optional<MessagingGuide>` | yes | MessageStrategist output; null until `MessagingGuideAttached` |
| `channelPlan` | `Optional<ChannelPlan>` | yes | ChannelPlanner output; null until `ChannelPlanAttached` |
| `assembled` | `Optional<AssembledBrief>` | yes | Director's assembled brief; null until `BriefAssembled` |
| `failureReason` | `Optional<String>` | yes | Set on `BriefBlocked` or `BriefDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `CampaignEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Brief creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `CampaignView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record ObjectiveRequest(String objective, String requestedBy) {}

record StrategyPlan(String marketQuery, String audienceQuestion,
                   String messagingBrief, String channelDirective) {}

record MarketInsight(String headline, String source, String detail) {}
record MarketInsightsBundle(List<MarketInsight> insights, Instant gatheredAt) {}

record Segment(String name, String description, List<String> characteristics) {}
record AudienceProfile(List<Segment> segments, String primaryPersona, Instant profiledAt) {}

record MessagingGuide(String coreMessage, List<String> supportingMessages,
                     String toneGuidance, Instant createdAt) {}

record ChannelRecommendation(String channel, String priority, String justification) {}
record ChannelPlan(List<ChannelRecommendation> channels, String rationale, Instant createdAt) {}

record AssembledBrief(String executiveSummary, MarketInsightsBundle marketInsights,
                     AudienceProfile audienceProfile, MessagingGuide messagingGuide,
                     ChannelPlan channelPlan, String complianceVerdict, Instant assembledAt) {}
```

## Status enum

```java
enum BriefStatus { PLANNING, IN_PROGRESS, ASSEMBLED, DEGRADED, BLOCKED }
```

## Events

### CampaignBriefEntity

| Event | Trigger |
|---|---|
| `BriefCreated` | Workflow creates the brief (`createBrief`) |
| `MarketInsightsAttached` | MarketResearcher returns a `MarketInsightsBundle` |
| `AudienceProfileAttached` | AudienceTargeter returns an `AudienceProfile` |
| `MessagingGuideAttached` | MessageStrategist returns a `MessagingGuide` |
| `ChannelPlanAttached` | ChannelPlanner returns a `ChannelPlan` |
| `BriefAssembled` | Director assembly passes the compliance guardrail |
| `BriefDegraded` | At least one worker timed out; assembled from partial input |
| `BriefBlocked` | Guardrail rejected the assembled brief |
| `CampaignEvalScored` | `EvalSampler` recorded a 1–5 quality score |

### ObjectiveQueue

| Event | Trigger |
|---|---|
| `ObjectiveSubmitted` | `enqueueObjective(objective, requestedBy)` from endpoint or simulator |

Fields: `{ briefId, objective, requestedBy, submittedAt }`.
