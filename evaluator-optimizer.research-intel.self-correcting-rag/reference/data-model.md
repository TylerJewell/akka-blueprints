# Data model — self-correcting-rag

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `QueryRequest` | `queryText` | `String` | no | The user-submitted research question. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `Document` | `docId` | `String` | no | Unique document identifier. |
| | `title` | `String` | no | Document title. |
| | `content` | `String` | no | Full document text. |
| | `source` | `String` | no | Provenance label (e.g., `"corpus"`, `"web-search"`). |
| `DocumentGrade` | `docId` | `String` | no | Matches the graded document. |
| | `verdict` | `RelevanceVerdict` | no | `RELEVANT` or `IRRELEVANT`. |
| | `score` | `double` | no | 0–1 relevance score. |
| | `rationale` | `String` | no | One sentence explaining the score. |
| `RetrievalResult` | `queryUsed` | `String` | no | The query string issued to the corpus. |
| | `documents` | `List<Document>` | no | Candidate documents, up to topK. |
| | `retrievedAt` | `Instant` | no | When the retrieval completed. |
| `GradingResult` | `grades` | `List<DocumentGrade>` | no | One grade per retrieved document. |
| | `retainedDocuments` | `List<Document>` | no | Subset of documents that are `RELEVANT`. |
| | `totalRetrieved` | `int` | no | Count of input documents. |
| | `gradedAt` | `Instant` | no | When grading completed. |
| `RewrittenQuery` | `originalQuery` | `String` | no | The query that produced no relevant docs. |
| | `rewrittenQueryText` | `String` | no | The new query for the next retrieval pass. |
| | `rewriteRationale` | `String` | no | One sentence explaining the rewrite. |
| | `rewrittenAt` | `Instant` | no | When the rewrite completed. |
| `GeneratedAnswer` | `answerText` | `String` | no | The answer, 2–5 sentences. |
| | `citedDocIds` | `List<String>` | no | Document IDs cited in the answer. |
| | `generatedAt` | `Instant` | no | When the answer was generated. |
| `HallucinationVerdict` | `result` | `HallucinationResult` | no | `GROUNDED` or `HALLUCINATED`. |
| | `rationale` | `String` | no | One sentence summarising the finding. |
| | `unsupportedClaims` | `List<String>` | no | Claims not supported by retained docs; empty when `GROUNDED`. |
| | `gradedAt` | `Instant` | no | When the hallucination check completed. |
| `RetrievalPass` | `passNumber` | `int` | no | 1-indexed; monotonic across rewrite cycles. |
| | `retrieval` | `RetrievalResult` | no | The retriever's output for this pass. |
| | `grading` | `GradingResult` | no | The relevance grader's output for this pass. |
| | `rewrite` | `Optional<RewrittenQuery>` | yes | Populated when this pass triggered a rewrite. |
| `Query` (entity state) | `queryId` | `String` | no | Unique id. |
| | `queryText` | `String` | no | Original research question. |
| | `requestedBy` | `String` | no | Submitter identifier. |
| | `maxRewriteAttempts` | `int` | no | Rewrite ceiling (default 2). |
| | `maxGenerationAttempts` | `int` | no | Generation retry ceiling (default 2). |
| | `status` | `QueryStatus` | no | See enum. |
| | `retrievalPasses` | `List<RetrievalPass>` | no | All retrieval passes; starts empty. |
| | `generatedAnswer` | `Optional<GeneratedAnswer>` | yes | Populated on `AnswerGenerated`. |
| | `hallucinationVerdict` | `Optional<HallucinationVerdict>` | yes | Populated on `HallucinationChecked`. |
| | `failureReason` | `Optional<String>` | yes | Populated on failure terminal events. |
| | `createdAt` | `Instant` | no | When `QueryCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the query reached a terminal state. |

## Enums

`QueryStatus`: `RETRIEVING`, `GRADING`, `REWRITING`, `GENERATING`, `HALLUCINATION_CHECK`, `ANSWERED`, `FAILED_NO_RELEVANT_DOCS`, `FAILED_HALLUCINATION`.

`RelevanceVerdict`: `RELEVANT`, `IRRELEVANT`.

`HallucinationResult`: `GROUNDED`, `HALLUCINATED`.

## Events (`QueryEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `QueryCreated` | `queryId, queryText, requestedBy, maxRewriteAttempts, maxGenerationAttempts, createdAt` | Workflow `startStep` | → `RETRIEVING` |
| `RetrievalPassStarted` | `passNumber, queryUsed` | Before `retrieveStep` for each pass | (no status change; tracks active pass) |
| `DocumentsRetrieved` | `passNumber, retrieval: RetrievalResult` | After `retrieveStep` returns | → `GRADING` |
| `DocumentsGraded` | `passNumber, grading: GradingResult` | After `gradeRelevanceStep` returns | `hasRelevant=true` → `GENERATING`; `hasRelevant=false, rewrites < max` → `REWRITING`; `hasRelevant=false, ceiling hit` → `FAILED_NO_RELEVANT_DOCS` |
| `QueryRewritten` | `passNumber, rewrite: RewrittenQuery` | After `rewriteStep` returns | → `RETRIEVING` (retry) |
| `AnswerGenerated` | `generationAttempt, answer: GeneratedAnswer` | After `generateStep` returns | → `HALLUCINATION_CHECK` |
| `HallucinationChecked` | `generationAttempt, verdict: HallucinationVerdict` | After `hallucinationCheckStep` returns | `GROUNDED` → `ANSWERED`; `HALLUCINATED, attempt < max` → `GENERATING`; `HALLUCINATED, ceiling hit` → `FAILED_HALLUCINATION` |
| `QueryAnswered` | `finalAnswer: GeneratedAnswer, gradedAt` | `HallucinationResult = GROUNDED` | → `ANSWERED`, `finishedAt = now` |
| `QueryFailedNoRelevantDocs` | `failureReason, passCount` | `passes.size() == maxRewriteAttempts` AND all grades `IRRELEVANT`; OR `defaultStepRecovery` failover | → `FAILED_NO_RELEVANT_DOCS`, `finishedAt = now` |
| `QueryFailedHallucination` | `failureReason, generationAttempt` | `generationAttempts == maxGenerationAttempts` AND verdict `HALLUCINATED`; OR `defaultStepRecovery` failover | → `FAILED_HALLUCINATION`, `finishedAt = now` |
| `EvalRecorded` | `passNumber, stepKind, verdict, score, docCount, recordedAt` | `EvalSampler` per grading step; workflow on terminal transition | (no status change; used for quality metrics) |

## Events (`CorpusEntity`)

| Event | Payload |
|---|---|
| `DocumentAdded` | `docId, title, content, source, addedAt` |

## View row

`QueryRow` is structurally identical to `Query` — the `retrievalPasses` list is bounded at `maxRewriteAttempts` (default 2) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `QueryEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllQueries` returning the full list. Callers filter by `status` client-side.
