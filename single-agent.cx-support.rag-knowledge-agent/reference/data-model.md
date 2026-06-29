# Data model — rag-knowledge-agent

Every record the generated system defines. Nullable lifecycle fields are `Optional<T>` (Lesson 6); list fields default to empty, never null.

## QuerySession (entity state + View row type)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| id | String | no | Session id (UUID) |
| question | String | no | The user's question |
| status | QueryStatus | no | Lifecycle state |
| receivedAt | Instant | no | When the question arrived |
| retrievedChunks | List<Citation> | no (empty default) | Passages retrieved for this question |
| retrievedAt | Optional<Instant> | yes | When retrieval finished |
| answer | Optional<String> | yes | The grounded answer |
| citations | List<Citation> | no (empty default) | Passages the answer cited |
| grounded | Optional<Boolean> | yes | Whether the answer was supported |
| answeredAt | Optional<Instant> | yes | When the answer was recorded |
| refusalReason | Optional<String> | yes | Why an answer was refused |
| faithfulnessScore | Optional<Double> | yes | Eval score in [0,1] |
| faithfulnessVerdict | Optional<String> | yes | supported / partial / unsupported |
| evaluatedAt | Optional<Instant> | yes | When the eval was recorded |

`emptyState()` returns `QuerySession.initial("")` with no `commandContext()` reference (Lesson 3).

## Citation

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| docId | String | no | Source document id |
| docTitle | String | no | Source document title |
| chunkId | String | no | Chunk id within the document |
| snippet | String | no | The passage text |
| score | double | no | Retrieval similarity score |

## Chunk (DocIndexEntity state element)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| docId | String | no | Source document id |
| docTitle | String | no | Source document title |
| chunkId | String | no | Chunk id |
| text | String | no | Chunk text |
| embedding | List<Double> | no | Deterministic lexical embedding |

`DocIndex(List<Chunk> chunks)` is the `DocIndexEntity` state; one well-known instance id `"default"`.

## Agent I/O records

- `AnswerRequest(String question, List<Citation> chunks)`
- `GroundedAnswer(String answer, List<Citation> citations, boolean grounded, String refusalReason)`
- `ScoreRequest(String answer, List<Citation> chunks)`
- `FaithfulnessResult(double score, String verdict)`
- `SearchQuery(List<Double> queryEmbedding, int topK)`

## Events (QuerySessionEntity)

| Event | Trigger |
|---|---|
| QueryReceived | `receive(question)` — a new question arrives |
| ChunksRetrieved | `recordRetrieval(chunks)` — retrieval completed |
| QueryAnswered | `recordAnswer(answer, citations)` — agent produced a grounded answer |
| QueryRefused | `recordRefusal(reason)` — grounding guardrail forced a refusal |
| FaithfulnessEvaluated | `recordEvaluation(score, verdict)` — eval completed |

## QueryStatus enum

`RECEIVED, RETRIEVED, ANSWERED, REFUSED, EVALUATED`
