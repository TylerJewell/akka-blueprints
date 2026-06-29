# Data model — spec-to-pr

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Requirement` | `reqId` | `String` | no | Short stable id (`r-<8 hex>`). |
| | `text` | `String` | no | Requirement sentence extracted from the spec. |
| | `priority` | `String` | no | `high`, `medium`, or `low`. |
| | `affectedFiles` | `List<String>` | no | File paths from `identifyAffectedFiles`. Possibly empty. |
| `ParsedSpec` | `specTitle` | `String` | no | First line or inferred title of the spec. |
| | `requirements` | `List<Requirement>` | no | Possibly empty; J6-style empty-spec path uses this. |
| | `parsedAt` | `Instant` | no | When the PARSE task returned. |
| `FileChange` | `filePath` | `String` | no | MUST appear in the codebase manifest. |
| | `changeType` | `String` | no | `add`, `modify`, or `delete`. |
| | `rationale` | `String` | no | One sentence explaining why this file is changed. |
| | `proposedDiff` | `String` | no | Unified-diff fragment; MUST NOT exceed 200 lines (C1 lint rule). |
| `ChangePlan` | `changes` | `List<FileChange>` | no | Possibly empty. |
| | `plannedAt` | `Instant` | no | When the PLAN task returned. |
| `DraftPr` | `title` | `String` | no | ≤ 72 characters. |
| | `description` | `String` | no | Markdown body. |
| | `fileChanges` | `List<FileChange>` | no | Same list as `ChangePlan.changes`. |
| | `draftedAt` | `Instant` | no | When the DRAFT task returned. |
| `CiCheck` | `name` | `String` | no | `compile`, `test`, or `lint`. |
| | `passed` | `boolean` | no | True if the check passed. |
| | `message` | `String` | no | One sentence — pass confirmation or failure description. |
| `CiResult` | `passed` | `boolean` | no | True iff all `checks[i].passed`. |
| | `checks` | `List<CiCheck>` | no | Exactly 3 entries (compile, test, lint). |
| | `evaluatedAt` | `Instant` | no | When `CiScorer` finished. |
| `ReviewDecision` | `reviewerId` | `String` | no | Identity of the reviewer. |
| | `decision` | `String` | no | `approved` or `rejected`. |
| | `comment` | `String` | no | Free-text comment from the reviewer. |
| | `decidedAt` | `Instant` | no | When the reviewer submitted the decision. |
| `GuardrailRejection` | `phase` | `String` | no | `PARSE` / `PLAN` / `DRAFT`. |
| | `tool` | `String` | no | Name of the misordered tool. |
| | `reason` | `String` | no | Structured reason from `WriteGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `SpecRunRecord` (entity state) | `runId` | `String` | no | — |
| | `specText` | `Optional<String>` | yes | Populated after `RunCreated`. |
| | `parsedSpec` | `Optional<ParsedSpec>` | yes | Populated after `SpecParsed`. |
| | `changePlan` | `Optional<ChangePlan>` | yes | Populated after `PlanProduced`. |
| | `draftPr` | `Optional<DraftPr>` | yes | Populated after `PrDrafted`. |
| | `reviewDecision` | `Optional<ReviewDecision>` | yes | Populated after `ReviewApproved` or `ReviewRejected`. |
| | `ciResult` | `Optional<CiResult>` | yes | Populated after `CiPassed` or `CiFailed`. |
| | `status` | `SpecRunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RunCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `SpecRunRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`SpecRunStatus`: `CREATED`, `PARSING`, `PARSED`, `PLANNING`, `PLANNED`, `DRAFTING`, `DRAFTED`, `AWAITING_REVIEW`, `CI_RUNNING`, `CI_PASSED`, `MERGE_READY`, `CI_FAILED`, `REJECTED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `WriteGuardrail`): `PARSE`, `PLAN`, `DRAFT`.

## Events (`SpecRunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunCreated` | `specText: String` | → CREATED |
| `ParseStarted` | — | → PARSING |
| `SpecParsed` | `parsedSpec: ParsedSpec` | → PARSED |
| `PlanStarted` | — | → PLANNING |
| `PlanProduced` | `changePlan: ChangePlan` | → PLANNED |
| `DraftStarted` | — | → DRAFTING |
| `PrDrafted` | `draftPr: DraftPr` | → DRAFTED → AWAITING_REVIEW |
| `ReviewApproved` | `reviewDecision: ReviewDecision` | → CI_RUNNING |
| `ReviewRejected` | `reviewDecision: ReviewDecision` | → REJECTED (terminal) |
| `CiStarted` | — | → CI_RUNNING |
| `CiPassed` | `ciResult: CiResult` | → CI_PASSED → MERGE_READY (terminal happy) |
| `CiFailed` | `ciResult: CiResult` | → CI_FAILED (terminal) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `RunFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `SpecRunRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SpecRunRow` mirrors `SpecRunRecord` exactly. The UI fetches the full row via `GET /api/runs/{id}` and streams updates via `GET /api/runs/sse`.

The view declares ONE query: `getAllRuns: SELECT * AS runs FROM spec_run_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`SpecTasks.java`)

```java
public final class SpecTasks {
  public static final Task<ParsedSpec> PARSE_SPEC = Task
      .name("Parse spec")
      .description("Extract requirements and identify affected files from a product spec")
      .resultConformsTo(ParsedSpec.class);

  public static final Task<ChangePlan> PLAN_CHANGES = Task
      .name("Plan changes")
      .description("Propose file edits that satisfy the parsed requirements")
      .resultConformsTo(ChangePlan.class);

  public static final Task<DraftPr> DRAFT_PR = Task
      .name("Draft PR")
      .description("Compose a pull-request title and description grounded in the change plan")
      .resultConformsTo(DraftPr.class);

  private SpecTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `ParseTools`, `PlanTools`, and `DraftTools` carries a `Phase` constant. `WriteGuardrail` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix (see eval-matrix.yaml G1). The tool registry is built once at startup; the guardrail reads it for every call.
