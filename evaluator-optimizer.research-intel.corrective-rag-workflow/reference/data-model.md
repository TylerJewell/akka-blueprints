# Data model — corrective-rag-workflow

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `QueryRequest` | `question` | `String` | no | The user's research query string. |
| | `relevanceThresholdOverride` | `Optional<Double>` | yes | Per-query override of the default 0.7 threshold. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `ChunkScore` | `chunkId` | `String` | no | Identifier of the corpus chunk. |
| | `excerpt` | `String` | no | First 200 characters of the chunk content. |
| | `score` | `double` | no | Relevance score 0–1 assigned by the evaluator. |
| `RetrievalResult` | `chunks` | `List<ChunkScore>` | no | Retrieved and ranked chunks; empty if none found. |
| | `retrievedAt` | `Instant` | no | When the RetrieverAgent returned. |
| `RelevanceVerdict` | `judgment` | `RelevanceJudgment` | no | `SUFFICIENT` or `INSUFFICIENT`. |
| | `scoredChunks` | `List<ChunkScore>` | no | Evaluator-scored chunks. |
| | `averageScore` | `double` | no | Arithmetic mean of per-chunk scores. |
| | `rationale` | `String` | no | One-sentence explanation. |
| | `evaluatedAt` | `Instant` | no | When the RelevanceEvaluatorAgent returned. |
| `WebResult` | `url` | `String` | no | URL of the web page. |
| | `title` | `String` | no | Page title. |
| | `snippet` | `String` | no | Up to 200-character excerpt. |
| | `relevanceScore` | `double` | no | Estimated relevance 0–1. |
| `WebSearchResult` | `results` | `List<WebResult>` | no | Ranked web results; empty if search returned none. |
| | `searchQuery` | `String` | no | The exact string submitted to the search. |
| | `searchedAt` | `Instant` | no | When the WebSearchAgent returned. |
| `GuardrailVerdict` | `cleared` | `boolean` | no | Whether the guardrail allowed the tool call. |
| | `reasonCode` | `String` | no | `OK` or `DISALLOWED_PATTERN`. |
| | `detail` | `String` | no | Human-readable detail; empty when `cleared=true`. |
| `Answer` | `text` | `String` | no | The synthesized answer, 80–400 characters. |
| | `citedChunkIds` | `List<String>` | no | Chunk IDs cited inline; empty if none. |
| | `citedUrls` | `List<String>` | no | URLs cited inline; empty if none. |
| | `source` | `AnswerSource` | no | `RETRIEVAL_ONLY`, `RETRIEVAL_PLUS_WEB`, or `DEGRADED_RETRIEVAL`. |
| | `generatedAt` | `Instant` | no | When the AnswerSynthesizerAgent returned. |
| `RetrievalAttempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop. |
| | `retrieval` | `RetrievalResult` | no | What the RetrieverAgent returned. |
| | `verdict` | `RelevanceVerdict` | no | The evaluator's scoring of that retrieval. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Present only on the attempt where the guardrail ran (the first `INSUFFICIENT` attempt that reached the tool-call gate). |
| | `webSearch` | `Optional<WebSearchResult>` | yes | Present only when web search fired. |
| `Query` (entity state) | `queryId` | `String` | no | Unique id. |
| | `question` | `String` | no | Original query string. |
| | `relevanceThreshold` | `double` | no | Effective threshold for this query (default 0.7). |
| | `maxRetrievalAttempts` | `int` | no | Per-query ceiling (default 3). |
| | `status` | `QueryStatus` | no | See enum. |
| | `attempts` | `List<RetrievalAttempt>` | no | Bounded at `maxRetrievalAttempts`; starts empty. |
| | `answer` | `Optional<Answer>` | yes | Populated on `QueryAnswered` or `QueryDegraded`. |
| | `degradationReason` | `Optional<String>` | yes | Populated on `QueryDegraded`. |
| | `createdAt` | `Instant` | no | When `QueryCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the query reached a terminal state. |

## Enums

`QueryStatus`: `RETRIEVING`, `EVALUATING`, `WEB_SEARCHING`, `ANSWERED`, `ANSWER_DEGRADED`.

`RelevanceJudgment`: `SUFFICIENT`, `INSUFFICIENT`.

`AnswerSource`: `RETRIEVAL_ONLY`, `RETRIEVAL_PLUS_WEB`, `DEGRADED_RETRIEVAL`.

## Events (`QueryEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `QueryCreated` | `question, relevanceThreshold, maxRetrievalAttempts, createdAt` | Workflow `startStep` | → `RETRIEVING` |
| `RetrievalAttempted` | `attemptNumber, retrieval: RetrievalResult` | After `retrieveStep` returns | (no status change; appends to `attempts[]`) |
| `RelevanceVerdictRecorded` | `attemptNumber, verdict: RelevanceVerdict` | After `evaluateStep` returns | `SUFFICIENT` → `EVALUATING` (proceed to synthesize); `INSUFFICIENT` → `EVALUATING` (route to guardrail or re-retrieve) |
| `GuardrailVerdictRecorded` | `attemptNumber, guardrail: GuardrailVerdict` | After `guardrailStep` (in-process) | `cleared=true` → `WEB_SEARCHING`; `cleared=false` → `ANSWER_DEGRADED` |
| `WebSearchCompleted` | `attemptNumber, webSearch: WebSearchResult` | After `webSearchStep` returns | (no status change; populates `attempts[n].webSearch`) |
| `AnswerGenerated` | `answer: Answer` | After `synthesizeStep` returns | (no status change; populates `query.answer`) |
| `QueryAnswered` | `answerSource` | After `AnswerGenerated` on non-degraded path | → `ANSWERED`, `finishedAt = now` |
| `QueryDegraded` | `degradationReason, bestExcerpt` | Guardrail blocked OR `defaultStepRecovery` failover | → `ANSWER_DEGRADED`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, judgment, averageScore, webSearchFired, recordedAt` | `EvalSampler` per completed verdict; workflow on terminal transition | (no status change; appends to internal `evalEvents[]` view-side projection) |

## Events (`CorpusStore`)

| Event | Payload |
|---|---|
| `ChunkAdded` | `chunkId, content, keywords, addedAt` |

## View row

`QueryRow` is structurally identical to `Query` — the `attempts` list is bounded at `maxRetrievalAttempts` (default 3) so the row stays small enough to push down the SSE stream. The view's `TableUpdater` consumes every `QueryEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllQueries` returning the full list. Callers filter by `status` client-side.
