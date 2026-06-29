# Data model — fraud-flagging-team

Authoritative record, event, and enum definitions. `/akka:implement` writes these exactly. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

## FraudCase (CaseEntity state + CasesView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| id | `String` | no | Case UUID |
| customerId | `String` | no | Customer identifier |
| transactionRef | `Optional<String>` | yes | Transaction reference |
| amount | `Optional<Double>` | yes | Transaction amount |
| status | `CaseStatus` | no | Lifecycle state |
| openedAt | `Optional<Instant>` | yes | When the case opened |
| fraudScore | `Optional<Double>` | yes | Fraud-scoring worker output 0.0–1.0 |
| fraudSignals | `Optional<String>` | yes | Fraud-scoring worker signal text |
| compliant | `Optional<Boolean>` | yes | Compliance worker result |
| complianceNotes | `Optional<String>` | yes | Compliance worker notes |
| riskTier | `Optional<String>` | yes | Risk worker tier LOW/MEDIUM/HIGH |
| riskRationale | `Optional<String>` | yes | Risk worker rationale |
| supervisorRationale | `Optional<String>` | yes | Supervisor verdict rationale |
| verdictConfidence | `Optional<Double>` | yes | Supervisor verdict confidence |
| flaggedAt | `Optional<Instant>` | yes | When flagged |
| clearedAt | `Optional<Instant>` | yes | When cleared |
| confirmedAt | `Optional<Instant>` | yes | When analyst confirmed |
| confirmedBy | `Optional<String>` | yes | Analyst id who confirmed |
| analystNote | `Optional<String>` | yes | Dismiss note |
| dismissedAt | `Optional<Instant>` | yes | When dismissed |
| escalatedAt | `Optional<Instant>` | yes | When escalated |
| actionedAt | `Optional<Instant>` | yes | When the account action ran |
| actionTaken | `Optional<String>` | yes | Description of the gated action |

`emptyState()` returns `FraudCase.initial("", "")` and never references `commandContext()` (Lesson 3).

## CaseStatus enum

`ANALYZING`, `FLAGGED`, `CLEARED`, `CONFIRMED`, `ACTIONED`, `DISMISSED`, `ESCALATED`.

## Events

| Event | Trigger |
|---|---|
| CaseOpened | TransactionConsumer / FraudEndpoint opens a case |
| AnalysisRecorded | Workflow writes the three worker outputs |
| CaseFlagged | Supervisor verdict flags the transaction |
| CaseCleared | Supervisor verdict clears the transaction |
| CaseConfirmed | Analyst confirms via API |
| CaseDismissed | Analyst dismisses via API |
| CaseEscalated | StuckCaseMonitor escalates a stale flagged case |
| CaseActioned | Gated account action runs |
| TransactionQueued | TransactionQueue records an inbound transaction |

## Records

| Record | Fields |
|---|---|
| TransactionInput | `String customerId`, `double amount`, `String transactionRef`, `String memo` |
| FraudScore | `double score`, `String signals` |
| ComplianceFinding | `boolean compliant`, `String notes` |
| CustomerRiskAssessment | `String tier`, `String rationale` |
| SupervisorVerdict | `boolean flag`, `double confidence`, `String rationale` |

## Companion

`FraudTasks.java` declares the `SYNTHESIZE` `Task<SupervisorVerdict>` constant the AutonomousAgent's `definition().capability(TaskAcceptance.of(SYNTHESIZE)...)` references (Lesson 7). `Redaction.java` masks card/account/customer identifiers before any transaction field is logged (control S1).
