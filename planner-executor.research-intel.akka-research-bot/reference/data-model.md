# Data model — akka-research-bot

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `JobRequest` | `question` | `String` | no | User-submitted research question. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `SearchQuery` | `kind` | `SearchKind` | no | Which fixture source to use (WEB/ACADEMIC/NEWS/PATENT). |
| | `query` | `String` | no | Focused search string. |
| | `expectedContribution` | `String` | no | One sentence on what this query adds to the research. |
| `ResearchPlan` | `question` | `String` | no | The original research question. |
| | `queries` | `List<SearchQuery>` | no | Ordered list of search queries (3–6). |
| | `notes` | `Optional<String>` | yes | Planner's notes on expected gaps or constraints. |
| `SearchResult` | `query` | `String` | no | Echo of the search query. |
| | `kind` | `SearchKind` | no | Kind of search executed. |
| | `ok` | `boolean` | no | True if the searcher found matching fixture content. |
| | `content` | `String` | no | Raw textual result (pre-sanitize). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `ResultEntry` | `attempt` | `int` | no | 1-based attempt count for this query. |
| | `query` | `String` | no | The query text. |
| | `kind` | `SearchKind` | no | Kind of search. |
| | `verdict` | `ResultVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / REDACTED. |
| | `scrubbedContent` | `String` | no | Sanitized result text. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `ResultLedger` | `entries` | `List<ResultEntry>` | no | Append-only. |
| `ReportSection` | `heading` | `String` | no | Section heading. |
| | `body` | `String` | no | 2–4 sentence body text. |
| | `citations` | `List<String>` | no | Bibliography entry strings cited in this section. |
| `ResearchReport` | `title` | `String` | no | 10–15 word descriptive title. |
| | `summary` | `String` | no | 60–120 word summary. |
| | `sections` | `List<ReportSection>` | no | 2–4 sections. |
| | `bibliography` | `List<String>` | no | Formatted citation strings, each traceable to a ResultEntry. |
| | `producedAt` | `Instant` | no | When the writer produced the report. |
| `ResearchJob` (entity state) | `jobId` | `String` | no | Unique id. |
| | `question` | `String` | no | Original research question. |
| | `status` | `JobStatus` | no | See enum. |
| | `plan` | `Optional<ResearchPlan>` | yes | Populated after `JobPlanned`. |
| | `results` | `Optional<ResultLedger>` | yes | Populated after first `ResultRecorded` or `QueryBlocked`. |
| | `report` | `Optional<ResearchReport>` | yes | Populated after `JobCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `JobFailed` / `JobFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `JobHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `JobCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the job reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `PlanAssessment` | (sealed interface) | — | — | Permits `Sufficient` and `NeedsMore(revisedPlan: ResearchPlan)`. |

## Enums

- `SearchKind` → `WEB`, `ACADEMIC`, `NEWS`, `PATENT`.
- `ResultVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `REDACTED`.
- `JobStatus` → `PLANNING`, `SEARCHING`, `WRITING`, `COMPLETED`, `FAILED`, `HALTED`, `STALE`.

## Events (`ResearchJobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `JobCreated` | `jobId, question, createdAt` | → PLANNING |
| `JobPlanned` | `plan: ResearchPlan` | → SEARCHING |
| `QueryDispatched` | `query: SearchQuery` | no status change |
| `QueryBlocked` | `attempt, query, blocker` | no status change; appends a `ResultEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `ResultRecorded` | `entry: ResultEntry` | no status change; appends to `results.entries`. |
| `PlanRevised` | `plan: ResearchPlan` | no status change; replaces `plan`. |
| `WritingStarted` | `(none)` | → WRITING |
| `ReportBlocked` | `rejectionReason` | no status change; increments internal revision counter. |
| `JobCompleted` | `report: ResearchReport` | → COMPLETED, `finishedAt = now` |
| `JobFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `JobHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `JobFailedTimeout` | `failureReason` | → STALE, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`JobQueue`)

| Event | Payload |
|---|---|
| `JobSubmitted` | `jobId, question, requestedBy, submittedAt` |

## View row

`JobRow` mirrors `ResearchJob` minus the heavy result payload — `results.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `scrubbedContent` is capped at 240 characters. The UI fetches the full job by id on click via `GET /api/jobs/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
