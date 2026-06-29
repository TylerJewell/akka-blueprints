# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## AgentRun (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `runId` | `String` | no | UUID, also the workflow id |
| `query` | `String` | no | The submitted research query |
| `status` | `RunStatus` | no | Lifecycle state |
| `plan` | `Optional<SearchPlan>` | yes | Manager's search plan; null until `PlanReady` |
| `searchResults` | `Optional<SearchResultBundle>` | yes | SearchAgent output; null until `SearchCompleted` |
| `trace` | `Optional<RunTrace>` | yes | Assembled Phoenix trace; null until `TraceAssembled` |
| `traceReport` | `Optional<TraceReport>` | yes | Inspector's analysis; null until `TraceInspected` |
| `answer` | `Optional<SynthesisedAnswer>` | yes | Final synthesised answer; null until `RunSynthesised` |
| `blockedUrl` | `Optional<String>` | yes | The URL that triggered the allow-list guardrail; set on `ToolCallBlocked` |
| `failureReason` | `Optional<String>` | yes | Set on `RunPartial` or `ToolCallBlocked` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `RunEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `createdAt` | `Instant` | no | Run creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `RunView` row type mirrors this with `Optional<T>` on every nullable field. Token totals from `RunTrace` are promoted to the view row as `Optional<Integer> totalTokens` and `Optional<Long> totalDurationMs` for efficient UI rendering without deserialising nested spans.

## Supporting records

```java
record QueryRequest(String query, String requestedBy) {}

record SearchPlan(String searchQuery, int maxPages) {}

record PageResult(String url, String title, String excerpt, int tokensConsumed) {}

record SearchResultBundle(List<PageResult> results, int totalTokens, long durationMs) {}

record PhoenixSpan(String spanId, String stepName, String toolName,
                   int inputTokens, int outputTokens, long durationMs, Instant startedAt) {}

record RunTrace(List<PhoenixSpan> spans, int totalTokens, long totalDurationMs) {}

record TraceReport(String summary, String hotStep, Map<String, Integer> tokenBreakdown) {}

record SynthesisedAnswer(String answer, List<String> sources,
                         String guardrailVerdict, Instant synthesisedAt) {}
```

## Status enum

```java
enum RunStatus { DISPATCHED, SEARCHING, TRACED, PARTIAL, BLOCKED_TOOL }
```

## Events

### RunEntity

| Event | Trigger |
|---|---|
| `RunCreated` | Workflow creates the run (`createRun`) |
| `PlanReady` | ManagerAgent returns a `SearchPlan` |
| `SearchCompleted` | SearchAgent returns a `SearchResultBundle` |
| `TraceAssembled` | Workflow assembles `RunTrace` from `PhoenixSpan` records |
| `TraceInspected` | TraceInspector returns a `TraceReport` |
| `RunSynthesised` | ManagerAgent synthesis completes; guardrail passed |
| `RunPartial` | SearchAgent exceeded `maxPages`; synthesised from partial results |
| `ToolCallBlocked` | URL allow-list guardrail rejected a proposed fetch |
| `RunEvalScored` | `EvalSampler` recorded a 1–5 quality score |

### QueryQueue

| Event | Trigger |
|---|---|
| `QuerySubmitted` | `enqueueQuery(query, requestedBy)` from endpoint or simulator |

Fields: `{ runId, query, requestedBy, submittedAt }`.
