# Data model — sdlc-technical-designer

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TechConstraint` | `constraintId` | `String` | no | Stable id supplied by the user. |
| | `description` | `String` | no | What the constraint requires or forbids. |
| | `category` | `String` | no | One of `language`, `framework`, `infra`, `team-skill`, `compliance`. |
| `DesignRequest` | `designId` | `String` | no | UUID minted by `DesignEndpoint`. |
| | `featureTitle` | `String` | no | User-supplied short label. |
| | `featureDescription` | `String` | no | Full feature description body. Passed as task attachment. |
| | `projectContextKey` | `String` | no | References a `ProjectContext` profile. |
| | `constraints` | `List<TechConstraint>` | no | Submitted list (0–N). |
| | `requestedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ProjectContext` | `contextKey` | `String` | no | Lookup key matching `projectContextKey` on the request. |
| | `architecturePattern` | `String` | no | e.g. `microservices`, `monolith`, `event-driven`. |
| | `existingComponents` | `List<String>` | no | Components already present in the project. |
| | `preferredPatterns` | `List<String>` | no | e.g. `cqrs`, `saga`, `event-sourcing`. |
| | `targetLanguage` | `String` | no | e.g. `Java`. |
| `ComponentChoice` | `componentId` | `String` | no | Short camelCase identifier. |
| | `componentKind` | `String` | no | One of the allowed Akka primitive names. |
| | `rationale` | `String` | no | Why this primitive was selected. |
| | `dependsOn` | `List<String>` | no | ComponentIds this component reads from or writes to. |
| `DataField` | `fieldName` | `String` | no | Java field name. |
| | `fieldType` | `String` | no | Java type (e.g. `String`, `Optional<Instant>`). |
| | `required` | `boolean` | no | Whether the field is mandatory. |
| | `purpose` | `String` | no | One sentence; must be non-empty (scored by `ProposalScorer`). |
| `DataModelSketch` | `entityName` | `String` | no | Java class name for the entity or value object. |
| | `fields` | `List<DataField>` | no | At least 2 per entity. |
| | `eventTypes` | `List<String>` | no | Domain events emitted by this entity. |
| `ApiEndpointSketch` | `method` | `String` | no | HTTP verb or `SSE`. |
| | `path` | `String` | no | e.g. `/api/notifications/{id}`. |
| | `requestSummary` | `String` | no | Brief body or parameter description. |
| | `responseSummary` | `String` | no | Brief response description. |
| | `owningComponent` | `String` | no | ComponentId from the `components` list. |
| `DecisionLogEntry` | `decisionId` | `String` | no | Short kebab-case identifier. |
| | `componentId` | `String` | no | MUST match a `componentId` in the proposal's `components` list. |
| | `decision` | `String` | no | What was decided, one sentence. |
| | `rationale` | `String` | no | Why; 1-3 sentences; must be non-empty (scored by `ProposalScorer`). |
| | `alternativesConsidered` | `List<String>` | no | At least one entry per `DecisionLogEntry`. |
| `DesignProposal` | `components` | `List<ComponentChoice>` | no | One per Akka primitive selected. |
| | `dataModel` | `List<DataModelSketch>` | no | One per proposed entity. |
| | `apiSurface` | `List<ApiEndpointSketch>` | no | One per proposed endpoint. |
| | `decisionLog` | `List<DecisionLogEntry>` | no | One per significant design decision. |
| | `executiveSummary` | `String` | no | 2–4 sentences; must be non-empty. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `ProposalScorer` finished. |
| `DesignRequestState` (entity state) | `designId` | `String` | no | — |
| | `request` | `Optional<DesignRequest>` | yes | Populated after `DesignRequestSubmitted`. |
| | `context` | `Optional<ProjectContext>` | yes | Populated after `ContextLoaded`. |
| | `proposal` | `Optional<DesignProposal>` | yes | Populated after `ProposalRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `DesignStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `DesignRequestSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `DesignRequestState` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`DesignStatus`: `SUBMITTED`, `CONTEXT_LOADED`, `DESIGNING`, `PROPOSAL_RECORDED`, `EVALUATED`, `FAILED`.

Allowed `componentKind` values (enforced by `ProposalGuardrail`): `EventSourcedEntity`, `Workflow`, `HttpEndpoint`, `View`, `Consumer`, `AutonomousAgent`, `TimedAction`.

## Events (`DesignRequestEntity`)

| Event | Payload | Transition |
|---|---|---|
| `DesignRequestSubmitted` | `request` | → SUBMITTED |
| `ContextLoaded` | `context` | → CONTEXT_LOADED |
| `DesignStarted` | — | → DESIGNING |
| `ProposalRecorded` | `proposal` | → PROPOSAL_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `DesignFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `DesignRequestState.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`DesignRow` mirrors `DesignRequestState` minus `request.featureDescription` (the entity holds the full text; the view holds a truncated preview for the live list). The UI fetches the full description on demand via `GET /api/designs/{id}` and reads `request.featureDescription` from the JSON.

The view declares ONE query: `getAllDesigns: SELECT * AS designs FROM design_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`DesignTasks.java`)

```java
public final class DesignTasks {
  public static final Task<DesignProposal> DESIGN_FEATURE = Task
      .name("Design feature")
      .description("Read the attached feature description and produce a DesignProposal for the given project context")
      .resultConformsTo(DesignProposal.class);

  private DesignTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
