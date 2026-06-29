# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## AdvisorySession (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `sessionId` | `String` | no | UUID, also the workflow id |
| `companyName` | `String` | no | The startup's name |
| `sector` | `String` | no | Industry sector supplied in the profile |
| `status` | `SessionStatus` | no | Lifecycle state |
| `marketLandscape` | `Optional<MarketLandscape>` | yes | MarketResearcher output; null until `MarketLandscapeAttached` |
| `gtmStrategy` | `Optional<GtmStrategy>` | yes | GtmStrategist output; null until `GtmStrategyAttached` |
| `contentPlan` | `Optional<ContentPlan>` | yes | ContentPlanner output; null until `ContentPlanAttached` |
| `roadmap` | `Optional<ProductRoadmap>` | yes | RoadmapAdvisor output; null until `RoadmapAttached` |
| `report` | `Optional<AdvisoryReport>` | yes | Synthesised report; null until `ReportSynthesised` |
| `failureReason` | `Optional<String>` | yes | Set on `SessionBlocked` or `SessionDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `EvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Session creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `AdvisoryView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record StartupProfile(String companyName, String sector, String stage,
                      String problemStatement, String requestedBy) {}

record WorkItems(String marketQuery, String gtmBrief,
                 String contentBrief, String roadmapBrief) {}

record Competitor(String name, String positioning, String differentiator) {}

record MarketLandscape(List<Competitor> competitors,
                       List<String> targetSegments,
                       String marketSizeSummary,
                       Instant researchedAt) {}

record GtmStrategy(List<String> channels,
                   String pricingSignal,
                   List<String> launchSequence,
                   Instant strategyAt) {}

record ContentPillar(String theme, String rationale) {}

record ContentPlan(List<ContentPillar> pillars,
                   List<String> formats,
                   String cadenceRecommendation,
                   Instant plannedAt) {}

record RoadmapPhase(String name, String goal, List<String> milestones) {}

record ProductRoadmap(List<RoadmapPhase> phases,
                      String keyRisk,
                      Instant roadmappedAt) {}

record AdvisoryReport(String executiveSummary,
                      MarketLandscape marketLandscape,
                      GtmStrategy gtmStrategy,
                      ContentPlan contentPlan,
                      ProductRoadmap roadmap,
                      String guardrailVerdict,
                      Instant synthesisedAt) {}
```

## Status enum

```java
enum SessionStatus { PLANNING, IN_PROGRESS, SYNTHESISED, DEGRADED, BLOCKED }
```

## Events

### AdvisorySessionEntity

| Event | Trigger |
|---|---|
| `SessionCreated` | Workflow creates the session (`createSession`) |
| `MarketLandscapeAttached` | MarketResearcher returns a `MarketLandscape` |
| `GtmStrategyAttached` | GtmStrategist returns a `GtmStrategy` |
| `ContentPlanAttached` | ContentPlanner returns a `ContentPlan` |
| `RoadmapAttached` | RoadmapAdvisor returns a `ProductRoadmap` |
| `ReportSynthesised` | AdvisorSupervisor synthesis passes the guardrail |
| `SessionDegraded` | At least one specialist timed out; synthesised from partial inputs |
| `SessionBlocked` | Guardrail rejected the synthesised advisory report |
| `EvalScored` | `EvalSampler` recorded a 1–5 score |

### ProfileQueue

| Event | Trigger |
|---|---|
| `ProfileSubmitted` | `enqueueProfile(companyName, sector, stage, problemStatement, requestedBy)` from endpoint or simulator |

Fields: `{ sessionId, companyName, sector, stage, problemStatement, requestedBy, submittedAt }`.
