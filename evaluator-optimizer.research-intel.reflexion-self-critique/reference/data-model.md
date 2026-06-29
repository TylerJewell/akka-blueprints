# Data model — reflexion-self-critique

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ResearchQuestion` | `text` | `String` | no | The user-submitted research question. |
| | `citationFloor` | `int` | no | Minimum number of citations the answer must include. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `SourceDocument` | `docId` | `String` | no | Stable identifier for the document in the corpus. |
| | `title` | `String` | no | Human-readable title of the document. |
| | `snippet` | `String` | no | Relevant excerpt retrieved by the search tool. |
| | `url` | `String` | no | Source URL or reference path. |
| `CandidateAnswer` | `text` | `String` | no | The synthesized answer prose. |
| | `citations` | `List<SourceDocument>` | no | Retrieved sources used in the answer. |
| | `citationCount` | `int` | no | Size of `citations`; used by the guardrail. |
| | `answeredAt` | `Instant` | no | When the ActorAgent returned the answer. |
| `CitationGuardrailVerdict` | `passed` | `boolean` | no | Whether the citation floor was met. |
| | `reasonCode` | `String` | no | `OK` or `UNDER_CITED`. |
| | `detail` | `String` | no | Human-readable shortfall detail; empty when `passed=true`. |
| `ReflexionNote` | `reinforcementParagraph` | `String` | no | 2–4 sentence verbal memory the actor internalises on retry. |
| | `focusBullets` | `List<String>` | no | 0–3 specific actionable bullets (empty on `PASS`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Reflexion` | `verdict` | `ReflexionVerdict` | no | `PASS` or `RETRY`. |
| | `note` | `ReflexionNote` | no | The verbal reinforcement note. |
| | `score` | `int` | no | 1–5 rubric minimum across four dimensions. |
| | `reflectedAt` | `Instant` | no | When the ReflexionAgent returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked attempts). |
| | `answer` | `CandidateAnswer` | no | The ActorAgent's output for this attempt. |
| | `guardrail` | `CitationGuardrailVerdict` | no | The deterministic citation check's verdict. |
| | `reflexion` | `Optional<Reflexion>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `Query` (entity state) | `queryId` | `String` | no | Unique id. |
| | `questionText` | `String` | no | Original research question. |
| | `citationFloor` | `int` | no | Per-query minimum citation count. |
| | `maxAttempts` | `int` | no | Per-query retry ceiling (default 4). |
| | `status` | `QueryStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `resolvedAttemptNumber` | `Optional<Integer>` | yes | Populated on `QueryResolved`. |
| | `resolvedAnswerText` | `Optional<String>` | yes | Populated on `QueryResolved`. |
| | `exhaustionReason` | `Optional<String>` | yes | Populated on `QueryExhausted`. |
| | `createdAt` | `Instant` | no | When `QueryCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the query reached a terminal state. |

## Enums

`QueryStatus`: `RESEARCHING`, `REFLECTING`, `RESOLVED`, `EXHAUSTED`.

`ReflexionVerdict`: `PASS`, `RETRY`.

## Events (`QueryEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `QueryCreated` | `questionText, citationFloor, maxAttempts, createdAt` | Workflow `startStep` | → `RESEARCHING` |
| `AttemptAnswered` | `attemptNumber, answer: CandidateAnswer` | After `answerStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptCitationGuardrailRecorded` | `attemptNumber, verdict: CitationGuardrailVerdict` | After `guardrailStep` | `passed=true` → `REFLECTING`; `passed=false` → `RESEARCHING` (re-answer) |
| `AttemptReflected` | `attemptNumber, reflexion: Reflexion` | After `reflectStep` returns | (no status change; populates `attempts[n].reflexion`) |
| `QueryResolved` | `attemptNumber, resolvedAnswerText` | `Reflexion.verdict = PASS` | → `RESOLVED`, `finishedAt = now` |
| `QueryExhausted` | `bestAttemptNumber, bestAnswerText, exhaustionReason` | `attempts.size() == maxAttempts` AND last reflexion is `RETRY`; OR `defaultStepRecovery` failover | → `EXHAUSTED`, `finishedAt = now` |
| `ReflexionRecorded` | `attemptNumber, verdict, score, citationShortfall, recordedAt` | `ReflexionSampler` per reflected attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`QueryQueue`)

| Event | Payload |
|---|---|
| `QuestionSubmitted` | `queryId, questionText, citationFloor, submittedBy, submittedAt` |

## View row

`QueryRow` is structurally identical to `Query` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `QueryEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllQueries` returning the full list. Callers filter by `status` client-side.
