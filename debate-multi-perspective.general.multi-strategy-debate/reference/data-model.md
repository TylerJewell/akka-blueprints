# Data model — multi-strategy-workflow

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `QuerySubmission` | `question` | `String` | no | The raw question text. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `StrategyBrief` | `keywordBrief` | `String` | no | Coordinator's brief for the keyword strategy. |
| | `semanticBrief` | `String` | no | Coordinator's brief for the semantic strategy. |
| | `chainOfThoughtBrief` | `String` | no | Coordinator's brief for the chain-of-thought strategy. |
| `EvidenceItem` | `source` | `String` | no | Source identifier for the passage or reasoning step. |
| | `excerpt` | `String` | no | Short excerpt or step claim (under 60 words). |
| | `relevanceScore` | `double` | no | 0.0–1.0 relevance to the query. |
| `StrategyResult` | `strategy` | `String` | no | `KEYWORD` / `SEMANTIC` / `CHAIN_OF_THOUGHT`. |
| | `answer` | `String` | no | The strategy's direct answer. |
| | `confidence` | `double` | no | 0.0–1.0 confidence in the answer. |
| | `evidence` | `List<EvidenceItem>` | no | 2–4 evidence items. |
| | `completedAt` | `Instant` | no | When the strategy agent completed. |
| `SynthesizedAnswer` | `answer` | `String` | no | The synthesized authoritative answer. |
| | `summary` | `String` | no | 60–120 word reconciliation. |
| | `strategyResults` | `List<StrategyResult>` | no | The strategy results synthesized. |
| | `guardrailVerdict` | `String` | no | `"ok"` or `"blocked: <reason>"`. |
| | `synthesizedAt` | `Instant` | no | When the coordinator synthesized. |
| `AgreementVerdict` | `score` | `int` | no | 1–5 cross-strategy agreement score. |
| | `rationale` | `String` | no | One-sentence reason. |
| `ValidationResult` | `valid` | `boolean` | no | Whether the query passed validation. |
| | `reason` | `String` | no | Rejection reason if `valid` is false; empty string otherwise. |
| `Query` (entity state, View row source) | `queryId` | `String` | no | Unique id. |
| | `question` | `String` | no | The question text. |
| | `status` | `QueryStatus` | no | See enum. |
| | `keywordResult` | `Optional<StrategyResult>` | yes | Populated after KeywordResultAttached. |
| | `semanticResult` | `Optional<StrategyResult>` | yes | Populated after SemanticResultAttached. |
| | `chainOfThoughtResult` | `Optional<StrategyResult>` | yes | Populated after ChainOfThoughtResultAttached. |
| | `answer` | `Optional<SynthesizedAnswer>` | yes | Populated after AnswerSynthesized. |
| | `failureReason` | `Optional<String>` | yes | Populated on QueryRejected / QueryDegraded / QueryBlocked. |
| | `agreementScore` | `Optional<Integer>` | yes | Populated on AgreementScored. |
| | `agreementRationale` | `Optional<String>` | yes | Populated on AgreementScored. |
| | `createdAt` | `Instant` | no | When QueryCreated emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the query reached a terminal state. |

Every nullable lifecycle field is `Optional<T>` because the View materializer rejects a row record with a non-optional `null` field (Lesson 6).

## Enums

`QueryStatus`: `RECEIVED`, `RUNNING`, `SYNTHESIZED`, `DEGRADED`, `BLOCKED`, `REJECTED`.

## Events (`QueryEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `QueryCreated` | `queryId, question, createdAt` | Workflow `createStep` | → RECEIVED |
| `QueryRejected` | `queryId, failureReason` | Workflow `validateStep`, input guardrail fails | → REJECTED, `finishedAt = now` |
| `QueryStarted` | `queryId` | Workflow `decomposeStep`, after brief generated | → RUNNING |
| `KeywordResultAttached` | `strategyResult` | `KeywordSearchAgent` returned | (no status change; populates `keywordResult`) |
| `SemanticResultAttached` | `strategyResult` | `SemanticRetrievalAgent` returned | (no status change; populates `semanticResult`) |
| `ChainOfThoughtResultAttached` | `strategyResult` | `ChainOfThoughtAgent` returned | (no status change; populates `chainOfThoughtResult`) |
| `AnswerSynthesized` | `answer` | Coordinator synthesized; guardrail OK | → SYNTHESIZED, `finishedAt = now` |
| `QueryDegraded` | `answer(partial), failureReason` | A strategy agent timed out | → DEGRADED, `finishedAt = now` |
| `QueryBlocked` | `answer(rejected), failureReason` | Output guardrail rejected | → BLOCKED, `finishedAt = now` |
| `AgreementScored` | `score, rationale` | `EvalSampler` → `ConsistencyJudge` | (no status change; populates `agreementScore` + `agreementRationale`) |

## Events (`QueryQueue`)

| Event | Payload |
|---|---|
| `QueryReceived` | `queryId, question, submittedBy, receivedAt` |

The `question` carried on `QueryReceived` is passed to the workflow start command. The queue event is the audit log; deployers who must minimise retention can drop the `question` from the queue event and re-fetch from source.

## View row

`QueryRow` mirrors `Query` but truncates each strategy result's `evidence` list to a count to keep the SSE stream small. The UI fetches the full query by id on expand, which includes the full evidence items.
