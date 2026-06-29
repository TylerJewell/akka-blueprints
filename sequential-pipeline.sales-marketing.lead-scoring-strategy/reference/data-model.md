# Data Model — Lead Scoring Strategy

Every nullable lifecycle field is `Optional<T>` (Lesson 6); the View materializer rejects non-optional `null` fields. Akka serializes `Optional<T>` as the raw value or `null`.

## `Lead` (entity state and View row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Lead id (UUID) |
| `company` | `Optional<String>` | yes | Company name |
| `contactName` | `Optional<String>` | yes | Sanitized contact name (PII redacted) |
| `formResponses` | `Optional<String>` | yes | Sanitized raw form text (PII redacted) |
| `status` | `LeadStatus` | no | Lifecycle status |
| `intakeAt` | `Optional<Instant>` | yes | When intake completed |
| `intakeProfile` | `Optional<String>` | yes | Structured profile from IntakeAgent |
| `researchedAt` | `Optional<Instant>` | yes | When research completed |
| `researchBrief` | `Optional<String>` | yes | Brief from ResearchAgent |
| `scoredAt` | `Optional<Instant>` | yes | When scoring completed |
| `score` | `Optional<Integer>` | yes | 0–100 priority score |
| `scoreRationale` | `Optional<String>` | yes | One-line score rationale |
| `evalAt` | `Optional<Instant>` | yes | When the auto-eval ran |
| `evalScore` | `Optional<Integer>` | yes | Eval confidence 0–100 |
| `evalFlags` | `Optional<String>` | yes | `none` or comma-joined accuracy/fairness flags |
| `strategizedAt` | `Optional<Instant>` | yes | When the strategy was drafted |
| `engagementStrategy` | `Optional<String>` | yes | Tailored strategy from StrategyAgent |

`initial(String id, String company)` returns a `Lead` with `status = NEW` and every `Optional` empty.

## `LeadStatus` enum

`NEW · INTAKE · RESEARCHED · SCORED · COMPLETE`

## `LeadEvent` (sealed interface)

| Event | Trigger | Sets |
|---|---|---|
| `LeadIntake(leadId, company, contactName, profile, timestamp)` | `intakeStep` after sanitization | `INTAKE`, intakeProfile, intakeAt |
| `LeadResearched(leadId, brief, timestamp)` | `researchStep` | `RESEARCHED`, researchBrief, researchedAt |
| `LeadScored(leadId, score, rationale, timestamp)` | `scoreStep` | `SCORED`, score, scoreRationale, scoredAt |
| `ScoreEvaluated(leadId, evalScore, flags, timestamp)` | `ScoreEvalConsumer` on `LeadScored` | evalScore, evalFlags, evalAt (no status change) |
| `StrategyProduced(leadId, strategy, timestamp)` | `strategyStep` | `COMPLETE`, engagementStrategy, strategizedAt |

`InboundLeadQueue` emits `InboundLeadQueued(leadId, rawSource, company, contactName, timestamp)`.

## Auxiliary records

```java
record IntakeProfile(String company, String contactName, String profile) {}
record ResearchBrief(String brief) {}
record LeadScore(int score, String rationale) {}
record EngagementStrategy(String strategy) {}
```

The eval result is computed in plain Java by `ScoreEvalConsumer` (no auxiliary agent record) and carried by `ScoreEvaluated`.

## View row type

`LeadsView` uses `Lead` as its row type. One query `getAllLeads` (`SELECT * AS leads FROM leads_view`) with no `WHERE status` clause — status filtering happens client-side (Lesson 2). `streamAllLeads` backs the SSE endpoint.
