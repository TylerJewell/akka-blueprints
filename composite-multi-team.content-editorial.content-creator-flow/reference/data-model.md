# Data model

Every Java record the generated system defines. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Campaign (CampaignEntity state + CampaignsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| id | `String` | no | Campaign id = workflow id |
| topic | `Optional<String>` | yes | Submitted topic |
| status | `CampaignStatus` | no | Lifecycle state |
| researchReport | `Optional<String>` | yes | Research summary |
| blogPost | `Optional<String>` | yes | Blog title + body, serialized |
| linkedInPost | `Optional<String>` | yes | LinkedIn body |
| brandVerdict | `Optional<String>` | yes | `PASS` or `BLOCK` |
| brandNotes | `Optional<String>` | yes | Reviewer reason |
| qualityScore | `Optional<Double>` | yes | Evaluator score in `[0,1]` |
| qualityNotes | `Optional<String>` | yes | Evaluator notes |
| createdAt | `Optional<Instant>` | yes | Set on `TopicReceived` |
| researchedAt | `Optional<Instant>` | yes | Set on `ResearchCompleted` |
| draftedAt | `Optional<Instant>` | yes | Set on `ContentDrafted` |
| reviewedAt | `Optional<Instant>` | yes | Set on review pass/block |
| evaluatedAt | `Optional<Instant>` | yes | Set on `QualityEvaluated` |
| completedAt | `Optional<Instant>` | yes | Set on `CampaignCompleted` |
| blockedAt | `Optional<Instant>` | yes | Set on `BrandReviewBlocked` |
| blockReason | `Optional<String>` | yes | Reviewer reason when blocked |

`emptyState()` returns `Campaign.initial("")` with no `commandContext()` reference (Lesson 3).

## CampaignStatus enum

`RECEIVED`, `RESEARCHING`, `DRAFTING`, `REVIEWING`, `EVALUATING`, `COMPLETED`, `BLOCKED`.

## Agent / command records

| Record | Fields |
|---|---|
| `ResearchReport` | `String summary` |
| `BlogPost` | `String title`, `String body` |
| `LinkedInPost` | `String body` |
| `ReviewInput` | `String topic`, `String researchReport`, `String blogPost`, `String linkedInPost` |
| `BrandVerdict` | `boolean passed`, `String reason` |
| `EvalInput` | `String topic`, `String blogPost`, `String linkedInPost` |
| `QualityResult` | `double score`, `String notes` |

## Events

| Event | Trigger |
|---|---|
| `TopicReceived` | Workflow starts; topic recorded |
| `ResearchCompleted` | ResearchAgent returns a report |
| `ContentDrafted` | Blog and LinkedIn agents both return |
| `BrandReviewPassed` | BrandReviewer returns `passed=true` |
| `BrandReviewBlocked` | BrandReviewer returns `passed=false` |
| `QualityEvaluated` | QualityEvaluator returns a score |
| `CampaignCompleted` | Pipeline reaches its terminal success state |

## InboundTopicQueue

State holds the enqueued topics. One command `enqueueTopic(String topic)` emits `TopicEnqueued { topic }`. `TopicConsumer` subscribes and starts one `ContentWorkflow` per event.

## CampaignsView

Row type is `Campaign`. One query: `getAllCampaigns` → `SELECT * AS campaigns FROM campaigns_view`. No `WHERE status` clause (Lesson 2); status filtering is client-side in `ContentEndpoint`.
