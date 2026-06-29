# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## SupplyOrder (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `orderId` | `String` | no | UUID, also the workflow id |
| `sku` | `String` | no | Product identifier from the demand signal |
| `targetQuantity` | `int` | no | Units requested |
| `status` | `OrderStatus` | no | Lifecycle state |
| `stockAssessment` | `Optional<StockAssessment>` | yes | StockAnalyst output; null until `StockAssessmentAttached` |
| `routePlan` | `Optional<RoutePlan>` | yes | LogisticsPlanner output; null until `RoutePlanAttached` |
| `recommendation` | `Optional<SupplyRecommendation>` | yes | Merged recommendation; null until `OrderRecommended` |
| `failureReason` | `Optional<String>` | yes | Set on `OrderBlocked` or `OrderDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `RecommendationScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Order creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `SupplyOrderView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record DemandSignal(String sku, int targetQuantity, String requestedBy) {}

record WorkOrder(String stockQuery, String routingBrief) {}

record StockPosition(String sku, int onHand, int inTransit, int reorderPoint,
                     boolean stockOutRisk, Instant assessedAt) {}

record StockAssessment(List<StockPosition> positions, String overallRisk,
                       Instant assessedAt) {}

record RouteSegment(String origin, String destination, int transitDays,
                    String carrier, double costUsd) {}

record RoutePlan(List<RouteSegment> segments, int totalTransitDays,
                 double totalCostUsd, Instant plannedAt) {}

record SupplyRecommendation(String summary, StockAssessment stockAssessment,
                             RoutePlan routePlan, String guardrailVerdict,
                             Instant synthesisedAt) {}
```

## Status enum

```java
enum OrderStatus { PENDING, ANALYZING, RECOMMENDED, DEGRADED, BLOCKED }
```

## Events

### SupplyOrderEntity

| Event | Trigger |
|---|---|
| `OrderCreated` | Workflow creates the order (`createOrder`) |
| `StockAssessmentAttached` | StockAnalyst returns a `StockAssessment` |
| `RoutePlanAttached` | LogisticsPlanner returns a `RoutePlan` |
| `OrderRecommended` | Coordinator synthesis passes the guardrail |
| `OrderDegraded` | A worker timed out; synthesised from partial input |
| `OrderBlocked` | Guardrail rejected the synthesised recommendation |
| `RecommendationScored` | `FulfillmentEvalSampler` recorded a 1–5 score |

### OrderQueueEntity

| Event | Trigger |
|---|---|
| `DemandSignalSubmitted` | `enqueueSignal(sku, targetQuantity, requestedBy)` from endpoint or simulator |

Fields: `{ orderId, sku, targetQuantity, requestedBy, submittedAt }`.
