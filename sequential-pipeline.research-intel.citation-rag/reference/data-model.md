# Data model — citation-rag

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Passage` | `passageId` | `String` | no | Stable short id (`p-<8 hex>`). |
| | `source` | `String` | no | Short name of the originating document. |
| | `documentTitle` | `String` | no | Full document title (paper, article, or policy name). |
| | `text` | `String` | no | The retrieved passage text. |
| | `relevanceScore` | `double` | no | Retrieval relevance 0.0–1.0. |
| | `retrievedAt` | `Instant` | no | When the RETRIEVE phase recorded this passage. |
| `PassageSet` | `passages` | `List<Passage>` | no | Possibly empty; J5 demonstrates the empty path. |
| | `retrievedAt` | `Instant` | no | When the RETRIEVE task returned. |
| `Claim` | `claimId` | `String` | no | Short stable id (`cl-<8 hex>`). |
| | `text` | `String` | no | The factual claim, paraphrased from the passage text. |
| | `citedPassageId` | `String` | no | MUST equal a `Passage.passageId` in the upstream `PassageSet`. |
| | `confidence` | `double` | no | Attribution confidence 0.0–1.0. |
| `ClaimSet` | `claims` | `List<Claim>` | no | Possibly empty. |
| | `attributedAt` | `Instant` | no | When the ATTRIBUTE task returned. |
| `Citation` | `claimId` | `String` | no | MUST equal a `Claim.claimId` in the same `Answer`. |
| | `passageId` | `String` | no | MUST equal a `Passage.passageId` in the upstream `PassageSet`. |
| | `passageSnippet` | `String` | no | Quoted text from the cited passage. |
| `Answer` | `queryId` | `String` | no | Echoed from the entity. |
| | `queryText` | `String` | no | Echoed query text. |
| | `responseBody` | `String` | no | 2–4 sentence grounded response. |
| | `claims` | `List<Claim>` | no | Every claim MUST have a non-null `citedPassageId` (G1 rule). |
| | `citations` | `List<Citation>` | no | One `Citation` per distinct `claimId` in `claims`. |
| | `composedAt` | `Instant` | no | When the COMPOSE task returned. |
| `CitationScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `CoverageScorer` finished. |
| `GuardrailRejection` | `claimId` | `String` | no | The claim that failed citation validation. |
| | `reason` | `String` | no | Structured reason from `CitationGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `QueryRecord` (entity state) | `queryId` | `String` | no | — |
| | `queryText` | `Optional<String>` | yes | Populated after `QueryCreated`. |
| | `passages` | `Optional<PassageSet>` | yes | Populated after `PassagesRetrieved`. |
| | `claimSet` | `Optional<ClaimSet>` | yes | Populated after `ClaimsAttributed`. |
| | `answer` | `Optional<Answer>` | yes | Populated after `AnswerComposed`. |
| | `citationScore` | `Optional<CitationScore>` | yes | Populated after `CitationScored`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QueryCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `CitationGuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `QueryRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QueryStatus`: `CREATED`, `RETRIEVING`, `RETRIEVED`, `ATTRIBUTING`, `ATTRIBUTED`, `COMPOSING`, `COMPOSED`, `EVALUATED`, `FAILED`.

`Phase` (used by tool-registration metadata and `CitationGuardrail`): `RETRIEVE`, `ATTRIBUTE`, `COMPOSE`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QueryCreated` | `queryText: String` | → CREATED |
| `RetrieveStarted` | — | → RETRIEVING |
| `PassagesRetrieved` | `passages: PassageSet` | → RETRIEVED |
| `AttributeStarted` | — | → ATTRIBUTING |
| `ClaimsAttributed` | `claimSet: ClaimSet` | → ATTRIBUTED |
| `ComposeStarted` | — | → COMPOSING |
| `AnswerComposed` | `answer: Answer` | → COMPOSED |
| `CitationScored` | `citationScore: CitationScore` | → EVALUATED (terminal happy) |
| `CitationGuardrailRejected` | `claimId, reason, rejectedAt` | no status change (audit-only) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `QueryRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `QueryRecord` exactly. The UI fetches the full row via `GET /api/queries/{id}` and streams updates via `GET /api/queries/sse`.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`CitationTasks.java`)

```java
public final class CitationTasks {
  public static final Task<PassageSet> RETRIEVE_PASSAGES = Task
      .name("Retrieve passages")
      .description("Search and fetch relevant passages for the query from the in-process corpus")
      .resultConformsTo(PassageSet.class);

  public static final Task<ClaimSet> ATTRIBUTE_CLAIMS = Task
      .name("Attribute claims")
      .description("Extract factual claims from passages and link each claim to its source passage")
      .resultConformsTo(ClaimSet.class);

  public static final Task<Answer> COMPOSE_ANSWER = Task
      .name("Compose answer")
      .description("Draft a grounded answer whose every claim cites a retrieved passage")
      .resultConformsTo(Answer.class);

  private CitationTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Citation guardrail invariant

`CitationGuardrail` fires on the `after-llm-response` hook for `COMPOSE_ANSWER` tasks. It reads the candidate `Answer.claims` list and for each claim asserts:

1. `claim.citedPassageId != null`
2. `passageSet.passages().stream().anyMatch(p -> p.passageId().equals(claim.citedPassageId()))`

Both conditions must hold for every claim. The guardrail rejects the entire answer if any claim fails, naming the first violating `claimId` in the rejection reason. Acceptance requires all claims to pass both checks.
