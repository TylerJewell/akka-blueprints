# Data model

Every record the generated system defines. Nullable lifecycle fields are `Optional<T>` (Lesson 6); on the wire they serialize as the raw value or `null`.

## Brief — entity state + view row

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| id | `String` | no | Brief id (UUID) |
| request | `Optional<String>` | yes | The original research request |
| status | `BriefStatus` | no | Lifecycle state |
| sectors | `List<String>` | no | Planned sectors (may be empty before PLANNED) |
| findings | `List<SectorFinding>` | no | One per analyzed sector (may be empty) |
| plannedAt | `Optional<Instant>` | yes | When the plan was recorded |
| briefTitle | `Optional<String>` | yes | Title from the supervisor plan |
| analyzedAt | `Optional<Instant>` | yes | When the last sector finding landed |
| briefBody | `Optional<String>` | yes | Synthesized body before sanitization |
| synthesizedAt | `Optional<Instant>` | yes | When synthesis was recorded |
| groundingScore | `Optional<Double>` | yes | Grounding guardrail score (0.0–1.0) |
| blockedReason | `Optional<String>` | yes | Why the brief was blocked |
| blockedAt | `Optional<Instant>` | yes | When it was blocked |
| sanitizedBody | `Optional<String>` | yes | Body with disclaimer appended (the released text) |
| sanitizedAt | `Optional<Instant>` | yes | When sanitization ran |
| completedAt | `Optional<Instant>` | yes | When the brief completed |

`emptyState()` returns `Brief.initial("")` with empty lists and empty Optionals — no `commandContext()` reference (Lesson 3).

## BriefStatus enum

`PLANNED`, `ANALYZING`, `SYNTHESIZED`, `BLOCKED`, `SANITIZED`, `COMPLETED`. BLOCKED and COMPLETED are terminal.

## Supporting records

| Record | Fields |
|---|---|
| SectorFinding | `String sector, String summary, List<String> signals` |
| ResearchPlan | `String briefTitle, List<String> sectors` (SupervisorAgent PLAN output) |
| SectorAnalysis | `String sector, String summary, List<String> signals` (SectorAnalystAgent ANALYZE output) |
| SynthesizedBrief | `String title, String body, double groundingScore` (SupervisorAgent SYNTHESIZE output) |

## Events (BriefEntity)

| Event | Trigger | Carries |
|---|---|---|
| BriefPlanned | `recordPlan` after planStep | briefTitle, sectors, plannedAt |
| SectorAnalyzed | `recordSectorFinding` per sector in analyzeStep | SectorFinding, analyzedAt |
| BriefSynthesized | `recordSynthesis` when grounding clears the bar | briefBody, groundingScore, synthesizedAt |
| BriefBlocked | `recordBlock` when groundingScore < 0.6 | blockedReason, blockedAt |
| BriefSanitized | `recordSanitization` in sanitizeStep | sanitizedBody, sanitizedAt |
| BriefCompleted | `recordCompletion` after sanitization | completedAt |

## InboundRequestQueue

| Event | Trigger | Carries |
|---|---|---|
| InboundRequestQueued | `enqueueRequest` from endpoint or simulator | request string |

## View row

`BriefsView` stores one `Brief` per id (row type = the `Brief` record above). Single query `getAllBriefs` (`SELECT * AS briefs FROM briefs_view`); status filtering happens client-side in the endpoint (Lesson 2).
