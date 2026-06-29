# Data model

Every record the generated system defines. Lifecycle fields are `Optional<T>` (Lesson 6) so the `EngagementsView` materializer accepts the row before those fields are populated. Akka's Jackson config serializes `Optional<T>` as the raw value or `null`.

## `Engagement` — entity state and `EngagementsView` row type

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Engagement id (workflow id). |
| `brief` | `Optional<String>` | yes | The submitted engagement request. |
| `status` | `EngagementStatus` | no | Lifecycle state. |
| `route` | `Optional<String>` | yes | `DELEGATE` or `HANDOFF`. |
| `complexityScore` | `Optional<Double>` | yes | Coordinator's 0.0–1.0 stakes score. |
| `routingRationale` | `Optional<String>` | yes | Why the coordinator routed this way. |
| `routedAt` | `Optional<Instant>` | yes | When routing was recorded. |
| `assignedTo` | `Optional<String>` | yes | `junior` or `senior`. |
| `deliverableTitle` | `Optional<String>` | yes | Title of the brief or recommendation. |
| `deliverableContent` | `Optional<String>` | yes | Body of the deliverable. |
| `deliveredAt` | `Optional<Instant>` | yes | When the deliverable was recorded. |
| `evalScore` | `Optional<Double>` | yes | Routing-decision eval score. |
| `evalNotes` | `Optional<String>` | yes | Routing-eval explanation. |
| `complianceReviewedAt` | `Optional<Instant>` | yes | When compliance review was recorded. |
| `complianceReviewer` | `Optional<String>` | yes | Reviewer identity. |
| `complianceVerdict` | `Optional<String>` | yes | `PASS` or `FLAG`. |
| `complianceNotes` | `Optional<String>` | yes | Reviewer notes. |

```java
public static Engagement initial(String id) { /* all Optional.empty(), status RECEIVED */ }
public Engagement applyEvent(EngagementEvent e) { /* per-variant switch */ }
```

`initial(...)` takes no `commandContext()` reference (Lesson 3).

## `EngagementStatus` (enum)

`RECEIVED · ROUTED · RESEARCHING · CONSULTING · DELIVERED · COMPLIANCE_REVIEWED · FLAGGED`

## `EngagementEvent` (sealed interface)

| Event | Trigger | Sets |
|---|---|---|
| `EngagementReceived(id, brief, ts)` | `EngagementWorkflow` start / `recordReceived` | `brief`, status `RECEIVED`. |
| `EngagementRouted(id, route, complexityScore, rationale, ts)` | `routeStep` → `recordRouting` | `route`, `complexityScore`, `routingRationale`, `routedAt`, status `ROUTED`. |
| `ResearchDelivered(id, title, content, ts)` | `delegateStep` → `recordResearch` | deliverable fields, `assignedTo = junior`, status `DELIVERED`. |
| `RecommendationDelivered(id, title, content, ts)` | `handoffStep` → `recordRecommendation` | deliverable fields, `assignedTo = senior`, status `DELIVERED`. |
| `RoutingEvaluated(id, score, notes, ts)` | `RoutingEvalConsumer` → `recordEval` | `evalScore`, `evalNotes`. |
| `ComplianceReviewed(id, reviewer, verdict, notes, ts)` | `POST /compliance` → `recordCompliance` | compliance fields, status `COMPLIANCE_REVIEWED` or `FLAGGED`. |

```java
sealed interface EngagementEvent permits
  EngagementReceived, EngagementRouted, ResearchDelivered,
  RecommendationDelivered, RoutingEvaluated, ComplianceReviewed { Instant timestamp(); }
```

## Auxiliary records (in `application/`)

```java
record RoutingDecision(String route, double complexityScore, String rationale) {}
record ResearchBrief(String title, String content) {}
record ConsultingRecommendation(String title, String content) {}
record ComplianceReview(String reviewer, String verdict, String notes) {}
```

## `InboundRequestQueue` event

```java
record InboundRequestQueued(String brief, Instant timestamp) {}
```

## View row type

`EngagementsView` uses `Engagement` directly as its row type. One query: `getAllEngagements` → `SELECT * AS engagements FROM engagements_view`. No `WHERE status` filter (Lesson 2); callers filter client-side.
