# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## CampaignPlan (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `planId` | `String` | no | UUID, also the workflow id |
| `campaignName` | `String` | no | The submitted campaign name |
| `objective` | `String` | no | The submitted campaign objective |
| `status` | `PlanStatus` | no | Lifecycle state |
| `launchBrief` | `Optional<LaunchBrief>` | yes | WebsiteLauncher output; null until `LaunchBriefAttached` |
| `strategyFramework` | `Optional<StrategyFramework>` | yes | StrategyAdvisor output; null until `StrategyFrameworkAttached` |
| `synthesised` | `Optional<SynthesisedPlan>` | yes | Merged plan; null until `PlanSynthesised` |
| `failureReason` | `Optional<String>` | yes | Set on `PlanBlocked` or `PlanDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `PlanEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line brand-quality justification |
| `createdAt` | `Instant` | no | Plan creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `CampaignView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record CampaignBriefRequest(String campaignName, String objective, String requestedBy) {}
record WorkScope(String launchScope, String strategyBrief) {}
record LaunchItem(String task, String priority, String rationale) {}
record LaunchBrief(List<LaunchItem> launchItems, String channelRecommendation, Instant preparedAt) {}
record StrategyFramework(String positioningStatement, List<String> messagingPillars,
                         List<String> targetAudiences, Instant preparedAt) {}
record SynthesisedPlan(String executiveSummary, LaunchBrief launchBrief,
                       StrategyFramework strategyFramework,
                       String guardrailVerdict, Instant synthesisedAt) {}
```

## Status enum

```java
enum PlanStatus { PLANNING, IN_PROGRESS, SYNTHESISED, DEGRADED, BLOCKED }
```

## Events

### CampaignPlanEntity

| Event | Trigger |
|---|---|
| `PlanCreated` | Workflow creates the plan (`createPlan`) |
| `LaunchBriefAttached` | WebsiteLauncher returns a `LaunchBrief` |
| `StrategyFrameworkAttached` | StrategyAdvisor returns a `StrategyFramework` |
| `PlanSynthesised` | CampaignDirector synthesis passes the guardrail |
| `PlanDegraded` | A specialist timed out; synthesised from partial input |
| `PlanBlocked` | Brand-safety guardrail rejected the synthesised plan |
| `PlanEvalScored` | `EvalSampler` recorded a 1–5 brand-quality score |

### RequestQueue

| Event | Trigger |
|---|---|
| `CampaignBriefSubmitted` | `submitBrief(campaignName, objective, requestedBy)` from endpoint or simulator |

Fields: `{ planId, campaignName, objective, requestedBy, submittedAt }`.
