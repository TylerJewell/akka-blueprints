# Data model — Stock Analysis Team

Authoritative. `/akka:implement` writes these records exactly. Every field null before its transition is `Optional<T>` (Lesson 6).

## Domain records (agent I/O)

| Record | Fields | Notes |
|---|---|---|
| `NewsFindings` | `List<String> headlines`, `String sentiment` | sentiment ∈ {positive, neutral, negative} |
| `FilingFindings` | `String revenue`, `String margin`, `List<String> riskFactors` | figures as stated in the filing |
| `ResearchBundle` | `String ticker`, `NewsFindings news`, `FilingFindings filings`, `String summary` | passed to summary + recommendation agents |
| `Recommendation` | `String call`, `String rationale`, `String disclaimer` | call ∈ {BUY, HOLD, SELL}; disclaimer required |

## Analysis record (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | analysis id (workflow id) |
| `ticker` | `String` | no | the requested ticker |
| `status` | `AnalysisStatus` | no | lifecycle status enum |
| `newsFindings` | `Optional<NewsFindings>` | yes | set after researchStep |
| `filingFindings` | `Optional<FilingFindings>` | yes | set after researchStep |
| `summary` | `Optional<String>` | yes | set after summarizeStep |
| `recommendation` | `Optional<String>` | yes | BUY/HOLD/SELL, set after recommendStep |
| `rationale` | `Optional<String>` | yes | set after recommendStep |
| `disclaimer` | `Optional<String>` | yes | set after recommendStep |
| `sanitizedNotes` | `Optional<String>` | yes | redaction notes from sanitizeStep |
| `driftFlag` | `Optional<Boolean>` | yes | set by DriftMonitor |
| `reviewedBy` | `Optional<String>` | yes | reviewer id on clear/retract |
| `reviewNote` | `Optional<String>` | yes | reviewer note |
| `queuedAt` | `Optional<Instant>` | yes | |
| `researchedAt` | `Optional<Instant>` | yes | |
| `summarizedAt` | `Optional<Instant>` | yes | |
| `sanitizedAt` | `Optional<Instant>` | yes | |
| `issuedAt` | `Optional<Instant>` | yes | |
| `reviewedAt` | `Optional<Instant>` | yes | |
| `failedAt` | `Optional<Instant>` | yes | |

`emptyState()` returns `Analysis.initial("", "")` with no `commandContext()` reference (Lesson 3).

## AnalysisStatus enum

`QUEUED`, `RESEARCHING`, `SUMMARIZING`, `SANITIZING`, `RECOMMENDING`, `ISSUED_PENDING_REVIEW`, `CLEARED`, `RETRACTED`, `FAILED`.

## Events

| Event | Trigger |
|---|---|
| `AnalysisQueued` | workflow starts; ticker recorded |
| `ResearchCompleted` | news + filings agents return |
| `SummaryCompleted` | summary agent returns |
| `Sanitized` | sanitize step scrubs research/summary text |
| `RecommendationIssued` | recommendation agent returns and passes the guardrail; status → ISSUED_PENDING_REVIEW |
| `ReviewCleared` | reviewer clears |
| `ReviewRetracted` | reviewer retracts |
| `DriftFlagged` | DriftMonitor flags a skewed distribution |
| `AnalysisFailed` | a step fails after retries, or the guardrail rejects the recommendation |

## InboundRequestQueue

State: pending tickers. Event: `InboundRequestQueued(ticker)`. Drives `RequestConsumer`, which starts one `AnalysisWorkflow` per event.
