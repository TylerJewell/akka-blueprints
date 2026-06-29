# Data model — graphrag-assistant

Every record the generated system defines. Nullable lifecycle fields are
`Optional<T>` (Lesson 6); on the wire they serialize as the raw value or `null`.

## `Query` — QueryEntity state and QueriesView row

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| id | String | no | Query UUID |
| question | String | no | The user's question |
| status | QueryStatus | no | Lifecycle state |
| createdAt | Instant | no | When the query was received |
| scope | Optional\<String\> | yes | "local" or "global", set when answered |
| chunkCount | Optional\<Integer\> | yes | Chunks used by the agent |
| retrievedAt | Optional\<Instant\> | yes | When retrieval ran |
| answer | Optional\<String\> | yes | Answer text, set on ANSWERED |
| grounded | Optional\<Boolean\> | yes | Grounding verdict |
| citations | Optional\<List\<String\>\> | yes | Source doc-ids |
| answeredAt | Optional\<Instant\> | yes | When answered |
| blockedReason | Optional\<String\> | yes | Reason, set on BLOCKED |

`emptyState()` returns `Query.initial("","")` with no `commandContext()` reference
(Lesson 3).

## `QueryStatus` enum

`RECEIVED`, `ANSWERED`, `BLOCKED`.

## `Answer` — ResearchAgent return type

| Field | Type | Meaning |
|---|---|---|
| text | String | The answer, drawn only from retrieved chunks |
| scope | String | "local" or "global" |
| grounded | boolean | True if every claim is chunk-supported |
| citations | List\<String\> | Source doc-ids (1–3) |
| chunkCount | int | Chunks used |

## `RetrievalResult` — CorpusIndex search return type

| Field | Type | Meaning |
|---|---|---|
| chunks | List\<String\> | Retrieved text chunks (sanitized before prompt) |
| citations | List\<String\> | Source doc-ids for the chunks |

## `IndexState` — CorpusIndex state

| Field | Type | Meaning |
|---|---|---|
| built | boolean | True once the index is built |
| docCount | int | Documents indexed |
| entityCount | int | Entity-level chunks (local-search corpus) |
| communityCount | int | Community summaries (global-search corpus) |

`emptyState()` returns `IndexState(false,0,0,0)`.

## Events on QueryEntity

| Event | Trigger |
|---|---|
| QueryReceived(id, question, createdAt) | `receive(question)` command |
| AnswerRecorded(answer, scope, chunkCount, citations, answeredAt) | `recordAnswer(Answer)` after grounding passes |
| AnswerBlocked(reason, blockedAt) | `block(reason)` when grounding fails |
