# Data model — sequential-workflow

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `FieldResult` | `fieldName` | `String` | no | Name of the job spec field being validated. |
| | `status` | `String` | no | `"OK"`, `"MISSING"`, or `"INVALID"`. |
| | `note` | `String` | no | Empty when status is `OK`; describes the problem otherwise. |
| `ValidationResult` | `fieldResults` | `List<FieldResult>` | no | One entry per declared field. Possibly empty when job type has no declared fields. |
| | `valid` | `boolean` | no | True only when every `fieldResult.status == "OK"`. |
| | `validatedAt` | `Instant` | no | When the VALIDATE task returned. |
| `ResolvedParam` | `key` | `String` | no | Parameter key from the job spec or a derived default. |
| | `value` | `String` | no | Resolved value. |
| | `source` | `String` | no | `"validation"`, `"default"`, or `"inferred"`. |
| `ContextMap` | `params` | `List<ResolvedParam>` | no | Possibly empty when job has no parameters. |
| | `metadata` | `Map<String, String>` | no | Job-type-specific key/value pairs. |
| | `resolvedAt` | `Instant` | no | When the ENRICH task completed parameter resolution. |
| `EnrichedJob` | `validation` | `ValidationResult` | no | The upstream validated state. |
| | `context` | `ContextMap` | no | Resolved parameters and metadata. |
| | `enrichedAt` | `Instant` | no | When the ENRICH task returned. |
| `Artifact` | `artifactId` | `String` | no | Short stable id (`"a-<8 hex>"`). |
| | `name` | `String` | no | Human-readable artifact name. |
| | `kind` | `String` | no | `"intermediate"`, `"output"`, or `"log"`. |
| | `ref` | `String` | no | Internal reference (e.g., `"mem://..."` for in-process artifacts). |
| `StepResult` | `stepId` | `String` | no | Short stable id for this step. MUST be unique within a `JobOutput`. |
| | `stepName` | `String` | no | Human-readable step name. |
| | `outcome` | `String` | no | `"success"`, `"skipped"`, or `"partial"`. |
| | `artifacts` | `List<Artifact>` | no | Possibly empty for `"skipped"` steps. |
| `JobOutput` | `steps` | `List<StepResult>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `executedAt` | `Instant` | no | When the EXECUTE task returned. |
| `SummaryEntry` | `stepId` | `String` | no | MUST equal a `StepResult.stepId` from the upstream `JobOutput`. |
| | `heading` | `String` | no | Section heading. |
| | `body` | `String` | no | 2–3 sentences summarising the step outcome and artifacts. |
| | `artifactIds` | `List<String>` | no | Each entry MUST equal an `Artifact.artifactId` from that step's `StepResult.artifacts`. Non-empty (E1 rule 2) unless step outcome is `"skipped"`. |
| `JobSummary` | `title` | `String` | no | 1-line title. |
| | `outcomeStatement` | `String` | no | 1-sentence overall outcome. |
| | `entries` | `List<SummaryEntry>` | no | `entries.size() == output.steps.size()` (E1 rule 4). |
| | `summarizedAt` | `Instant` | no | When the SUMMARIZE task returned. |
| `QualityResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `QualityScorer` finished. |
| `JobSpec` | `jobName` | `String` | no | User-supplied job name. |
| | `jobType` | `String` | no | Must match a seeded job-type key or a custom string. |
| | `parameters` | `Map<String, String>` | no | Possibly empty. |
| `GuardrailRejection` | `phase` | `String` | no | `"VALIDATE"` / `"ENRICH"` / `"EXECUTE"` / `"SUMMARIZE"`. |
| | `tool` | `String` | no | Name of the misordered tool. |
| | `reason` | `String` | no | Structured reason from `StepGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `JobRecord` (entity state) | `jobId` | `String` | no | — |
| | `spec` | `Optional<JobSpec>` | yes | Populated after `JobCreated`. |
| | `validation` | `Optional<ValidationResult>` | yes | Populated after `JobValidated`. |
| | `enrichedJob` | `Optional<EnrichedJob>` | yes | Populated after `JobEnriched`. |
| | `output` | `Optional<JobOutput>` | yes | Populated after `JobExecuted`. |
| | `summary` | `Optional<JobSummary>` | yes | Populated after `JobSummarized`. |
| | `quality` | `Optional<QualityResult>` | yes | Populated after `QualityScored`. |
| | `status` | `JobStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `JobCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `JobRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`JobStatus`: `CREATED`, `VALIDATING`, `VALIDATED`, `ENRICHING`, `ENRICHED`, `EXECUTING`, `EXECUTED`, `SUMMARIZING`, `SUMMARIZED`, `EVALUATED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `StepGuardrail`): `VALIDATE`, `ENRICH`, `EXECUTE`, `SUMMARIZE`.

## Events (`JobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `JobCreated` | `spec: JobSpec` | → CREATED |
| `ValidateStarted` | — | → VALIDATING |
| `JobValidated` | `validation: ValidationResult` | → VALIDATED |
| `EnrichStarted` | — | → ENRICHING |
| `JobEnriched` | `enrichedJob: EnrichedJob` | → ENRICHED |
| `ExecuteStarted` | — | → EXECUTING |
| `JobExecuted` | `output: JobOutput` | → EXECUTED |
| `SummarizeStarted` | — | → SUMMARIZING |
| `JobSummarized` | `summary: JobSummary` | → SUMMARIZED |
| `QualityScored` | `quality: QualityResult` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `JobFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `JobRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`JobRow` mirrors `JobRecord` exactly. The UI fetches the full row via `GET /api/jobs/{id}` and streams updates via `GET /api/jobs/sse`.

The view declares ONE query: `getAllJobs: SELECT * AS jobs FROM job_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`WorkflowTasks.java`)

```java
public final class WorkflowTasks {
  public static final Task<ValidationResult> VALIDATE_JOB = Task
      .name("Validate job")
      .description("Check all job fields against the job type's declared constraints and return a ValidationResult")
      .resultConformsTo(ValidationResult.class);

  public static final Task<EnrichedJob> ENRICH_JOB = Task
      .name("Enrich job")
      .description("Resolve parameters and attach contextual metadata to produce an EnrichedJob")
      .resultConformsTo(EnrichedJob.class);

  public static final Task<JobOutput> EXECUTE_JOB = Task
      .name("Execute job")
      .description("Run each declared step and collect output artifacts to produce a JobOutput")
      .resultConformsTo(JobOutput.class);

  public static final Task<JobSummary> SUMMARIZE_JOB = Task
      .name("Summarize job")
      .description("Build a per-step SummaryEntry for every executed step and compose the outcome statement")
      .resultConformsTo(JobSummary.class);

  private WorkflowTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `ValidateTools`, `EnrichTools`, `ExecuteTools`, and `SummarizeTools` carries a `Phase` constant. `StepGuardrail` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix (see eval-matrix.yaml G1). The tool registry is built once at startup; the guardrail reads it for every call.
