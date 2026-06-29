# Data model — json-query-engine

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `QueryIntent` | `intentType` | `String` | no | One of `lookup`, `filter`, or `aggregate`. |
| | `targetEntity` | `String` | no | Short description of what the question is asking for. |
| | `candidateRootKeys` | `List<String>` | no | Top-level JSON keys most likely to contain the answer. |
| `ParsedQuestion` | `questionText` | `String` | no | The original question verbatim. |
| | `intent` | `QueryIntent` | no | Structured intent extracted by the PARSE task. |
| | `parsedAt` | `Instant` | no | When the PARSE task returned. |
| `PathMatch` | `pathExpression` | `String` | no | A valid JSON-path expression starting with `$`. |
| | `resolvedValue` | `String` | no | The scalar value at that path in the document. |
| | `depthLevel` | `int` | no | Nesting depth where the value was found. |
| `TraversalResult` | `matches` | `List<PathMatch>` | no | Possibly empty when no paths match. |
| | `docId` | `String` | no | The document id that was traversed. |
| | `traversedAt` | `Instant` | no | When the TRAVERSE task returned. |
| `Citation` | `pathExpression` | `String` | no | MUST equal a `PathMatch.pathExpression` from the `TraversalResult`. |
| | `value` | `String` | no | MUST equal a `PathMatch.resolvedValue` from the `TraversalResult`. |
| | `label` | `String` | no | Short human-readable label derived from the last path segment. |
| `QueryResult` | `answer` | `String` | no | Complete natural-language answer (≥ 20 chars when matches exist). |
| | `citations` | `List<Citation>` | no | Possibly empty when `TraversalResult.matches` is empty. |
| | `composedAt` | `Instant` | no | When the RESPOND task returned. |
| `AccuracyResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `AccuracyScorer` finished. |
| `GuardrailRejection` | `phase` | `String` | no | `PARSE` / `TRAVERSE` / `RESPOND`. |
| | `tool` | `String` | no | Name of the rejected tool call. |
| | `pathExpr` | `String` | no | The path expression that failed validation (empty string for phase-mismatch rejections). |
| | `reason` | `String` | no | Structured reason from `PathGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `QueryRecord` (entity state) | `queryId` | `String` | no | — |
| | `question` | `Optional<String>` | yes | Populated after `QueryCreated`. |
| | `docId` | `Optional<String>` | yes | Populated after `QueryCreated`. |
| | `parsed` | `Optional<ParsedQuestion>` | yes | Populated after `QuestionParsed`. |
| | `traversal` | `Optional<TraversalResult>` | yes | Populated after `PathsTraversed`. |
| | `result` | `Optional<QueryResult>` | yes | Populated after `ResponseComposed`. |
| | `accuracy` | `Optional<AccuracyResult>` | yes | Populated after `AccuracyScored`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QueryCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `QueryRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`QueryStatus`: `CREATED`, `PARSING`, `PARSED`, `TRAVERSING`, `TRAVERSED`, `RESPONDING`, `RESPONDED`, `EVALUATED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `PathGuardrail`): `PARSE`, `TRAVERSE`, `RESPOND`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QueryCreated` | `question: String, docId: String` | → CREATED |
| `ParseStarted` | — | → PARSING |
| `QuestionParsed` | `parsed: ParsedQuestion` | → PARSED |
| `TraverseStarted` | — | → TRAVERSING |
| `PathsTraversed` | `traversal: TraversalResult` | → TRAVERSED |
| `RespondStarted` | — | → RESPONDING |
| `ResponseComposed` | `result: QueryResult` | → RESPONDED |
| `AccuracyScored` | `accuracy: AccuracyResult` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `phase, tool, pathExpr, reason, rejectedAt` | no status change (audit-only) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `QueryRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `QueryRecord` exactly. The UI fetches the full row via `GET /api/queries/{id}` and streams updates via `GET /api/queries/sse`.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`QueryTasks.java`)

```java
public final class QueryTasks {
  public static final Task<ParsedQuestion> PARSE_QUESTION = Task
      .name("Parse question")
      .description("Identify query intent, target entity, and candidate root keys from the question and document schema")
      .resultConformsTo(ParsedQuestion.class);

  public static final Task<TraversalResult> TRAVERSE_DOCUMENT = Task
      .name("Traverse document")
      .description("Resolve path expressions against the JSON document to extract matching values")
      .resultConformsTo(TraversalResult.class);

  public static final Task<QueryResult> COMPOSE_RESPONSE = Task
      .name("Compose response")
      .description("Write a grounded natural-language answer citing matched path/value pairs from the traversal result")
      .resultConformsTo(QueryResult.class);

  private QueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `ParseTools`, `TraverseTools`, and `RespondTools` carries a `Phase` constant. `PathGuardrail` reads this constant before the tool body runs and applies the phase-status matrix (see eval-matrix.yaml G1). For `TraverseTools` methods that accept a path expression, the guardrail additionally runs structural validation (syntax, root-key, depth) before the document file is opened. The tool registry is built once at startup; the guardrail reads it for every call.

## Document schema

Each seeded document has a matching schema file at `src/main/resources/sample-data/schemas/<docId>.json`:

```json
{
  "docId": "saas-pricing-catalog",
  "rootKeys": ["plans", "pricing", "features"],
  "maxTraversalDepth": 5
}
```

`PathGuardrail` loads all schema files at startup and uses them for root-key and depth validation. A document whose schema file is absent from the classpath causes `PathGuardrail` to reject all path expressions for that document with a `"no schema registered for docId"` reason.
