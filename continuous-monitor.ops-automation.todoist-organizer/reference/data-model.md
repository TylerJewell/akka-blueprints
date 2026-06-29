# Data model — todoist-organizer

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TodoistTask` | `taskId` | `String` | no | Unique id from Todoist (or simulator). |
| | `content` | `String` | no | Task title. |
| | `description` | `String` | no | Task body; may be empty string. |
| | `projectId` | `Optional<String>` | yes | Existing project id, if already assigned. |
| | `labels` | `List<String>` | no | Existing labels; may be empty. |
| | `priority` | `String` | no | Todoist priority string: `"p1"` – `"p4"`. |
| | `fetchedAt` | `Instant` | no | When the poller fetched this task. |
| `ClassificationResult` | `targetProjectId` | `String` | no | Project id to assign (empty if confidence low). |
| | `targetProjectName` | `String` | no | Human-readable project name. |
| | `targetLabels` | `List<String>` | no | Labels to apply; may be empty. |
| | `priorityLevel` | `String` | no | `"p1"` – `"p4"`. |
| | `confidence` | `String` | no | `"high" \| "medium" \| "low"`. |
| | `reason` | `String` | no | One short sentence. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | true = proceed to update; false = skip. |
| | `reason` | `String` | no | Why allowed or blocked. |
| `UpdateRecord` | `targetProjectId` | `String` | no | Project written to Todoist. |
| | `targetProjectName` | `String` | no | Human-readable name. |
| | `appliedLabels` | `List<String>` | no | Labels written to Todoist. |
| | `priorityLevel` | `String` | no | Priority written to Todoist. |
| | `updatedAt` | `Instant` | no | When the Todoist write completed. |
| `EvalResult` | `score` | `int` | no | 1–5 composite score. |
| | `rationale` | `String` | no | One sentence. |
| `OrganizerTask` (entity state) | `taskId` | `String` | no | — |
| | `original` | `TodoistTask` | no | Captured once at registerTask. |
| | `classification` | `Optional<ClassificationResult>` | yes | Populated after TaskClassified. |
| | `guardrailVerdict` | `Optional<GuardrailVerdict>` | yes | Populated after GuardrailChecked. |
| | `update` | `Optional<UpdateRecord>` | yes | Populated after TaskUpdated. |
| | `evalScore` | `Optional<Integer>` | yes | Populated after EvalScored. |
| | `evalRationale` | `Optional<String>` | yes | Populated after EvalScored. |
| | `status` | `TaskStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When TaskFetched emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

## Enums

`TaskStatus`: `FETCHED`, `CLASSIFIED`, `GUARDRAIL_BLOCKED`, `UPDATED`, `SKIPPED`, `FAILED`.

## Events (`TodoistTaskEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TaskFetched` | `original` | → FETCHED |
| `TaskClassified` | `classification` | → CLASSIFIED |
| `GuardrailChecked` | `guardrailVerdict` | → GUARDRAIL_BLOCKED (if blocked) or stays CLASSIFIED |
| `TaskUpdated` | `update` | → UPDATED (terminal) |
| `TaskSkipped` | `reason` | → SKIPPED (terminal) |
| `TaskFailed` | `errorMessage` | → FAILED (terminal) |
| `EvalScored` | `score, rationale` | (no status change; populates evalScore + evalRationale) |

## Events (`TaskQueue`)

| Event | Payload |
|---|---|
| `TaskFetched` | `task` (raw TodoistTask from the simulator — used by the audit log) |

## View row

`OrganizerTaskRow` mirrors `OrganizerTask`. The UI fetches full task detail via `GET /api/organizer/{id}`.
