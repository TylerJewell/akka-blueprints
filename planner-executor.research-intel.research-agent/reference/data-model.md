# Data model — research-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `QueryRequest` | `query` | `String` | no | User-submitted research question. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `SearchLedger` | `context` | `String` | no | Plain-text summary of the research goal as understood by the planner. |
| | `knownFacts` | `List<String>` | no | Facts the planner already considers established. |
| | `missingFacts` | `List<String>` | no | Facts the planner still needs to retrieve. |
| | `plan` | `List<String>` | no | Ordered list of search steps (3–6). |
| | `currentDispatch` | `Optional<SearchDecision>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `SearchDecision` | `searcher` | `SearcherKind` | no | Which searcher runs the step. |
| | `query` | `String` | no | One-sentence search query. |
| | `rationale` | `String` | no | One-sentence justification for this choice. |
| `SearchResult` | `searcher` | `SearcherKind` | no | Searcher that ran the step. |
| | `query` | `String` | no | Echo of the query. |
| | `ok` | `boolean` | no | True if the searcher found relevant content. |
| | `content` | `String` | no | Raw textual result (pre-citation-eval). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `FindingEntry` | `attempt` | `int` | no | 1-based attempt count for this `(searcher, query)` pair. |
| | `searcher` | `SearcherKind` | no | Searcher that ran this step. |
| | `query` | `String` | no | The query text. |
| | `content` | `String` | no | The retrieved content. |
| | `citationVerdict` | `CitationVerdict` | no | CITED or UNCITED. |
| | `confidence` | `double` | no | Signal-match confidence in [0.0, 1.0]. |
| | `noSourceReason` | `Optional<String>` | yes | Populated when `citationVerdict = UNCITED`; names which signals were absent. |
| | `recordedAt` | `Instant` | no | When the finding was appended. |
| `FindingsLedger` | `entries` | `List<FindingEntry>` | no | Append-only. |
| `Citation` | `label` | `String` | no | Short human-readable label (e.g., "EU AI Act Article 13"). |
| | `source` | `String` | no | Source identifier (URL, document path, or author-year). |
| | `searcher` | `String` | no | Which searcher retrieved this citation. |
| `ResearchReport` | `title` | `String` | no | Report title (one sentence). |
| | `summary` | `String` | no | 60–120 word synthesis. |
| | `findings` | `List<String>` | no | Key findings, one per bullet. |
| | `citations` | `List<Citation>` | no | Cited sources; only entries with `citationVerdict = CITED`. |
| | `producedAt` | `Instant` | no | When the planner produced the report. |
| `ResearchJob` (entity state) | `jobId` | `String` | no | Unique id. |
| | `query` | `String` | no | Original query. |
| | `status` | `JobStatus` | no | See enum. |
| | `searchLedger` | `Optional<SearchLedger>` | yes | Populated after `JobPlanned`. |
| | `findingsLedger` | `Optional<FindingsLedger>` | yes | Populated after first `FindingRecorded`. |
| | `report` | `Optional<ResearchReport>` | yes | Populated after `JobCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `JobFailed` / `JobFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `JobHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `JobCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the job reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `NextStep` | (sealed interface) | — | — | Permits `Continue(SearchDecision)`, `Replan(SearchLedger revised)`, `Complete(ResearchReport stub)`, `Fail(String reason)`. |

## Enums

- `SearcherKind` → `WEB`, `DOCUMENT`.
- `CitationVerdict` → `CITED`, `UNCITED`.
- `JobStatus` → `PLANNING`, `SEARCHING`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`ResearchJobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `JobCreated` | `jobId, query, createdAt` | → PLANNING |
| `JobPlanned` | `searchLedger` | → SEARCHING |
| `StepDispatched` | `dispatch` | no status change; sets `searchLedger.currentDispatch`. |
| `StepRetried` | `attempt, dispatch` | no status change; increments attempt count for the `(searcher, query)` pair. |
| `FindingRecorded` | `entry: FindingEntry` | no status change; appends to `findingsLedger.entries`. |
| `LedgerRevised` | `searchLedger: SearchLedger` | no status change; replaces `searchLedger`. |
| `JobCompleted` | `report: ResearchReport` | → COMPLETED, `finishedAt = now` |
| `JobFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `JobHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `JobFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`QueryQueue`)

| Event | Payload |
|---|---|
| `QuerySubmitted` | `jobId, query, requestedBy, submittedAt` |

## View row

`JobRow` mirrors `ResearchJob` minus the heavy findings payload — `findingsLedger.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `content` is capped at 240 characters. The UI fetches the full job by id on click via `GET /api/jobs/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
