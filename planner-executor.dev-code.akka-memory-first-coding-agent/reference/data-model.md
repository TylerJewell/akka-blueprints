# Data model — akka-memory-first-coding-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InitRequest` | `projectPath` | `String` | no | Root path of the project to research. |
| | `requestedBy` | `String` | no | UI identifier of the user. |
| `ResearchPlan` | `filesToRead` | `List<String>` | no | Ordered list of 4–8 file paths to read. |
| | `questions` | `List<String>` | no | 3–6 questions to answer across the reading loop. |
| `FileInsight` | `fileName` | `String` | no | Path of the file that was read. |
| | `language` | `String` | no | Detected language identifier. |
| | `summary` | `String` | no | 2–4 sentence summary answering the assigned question. |
| | `keySymbols` | `List<String>` | no | 3–8 identifiers from the file (class names, config keys, route paths). |
| `MemoryBlock` | `name` | `String` | no | Short kebab-case identifier for the block. |
| | `content` | `String` | no | 3–8 sentence summary of this aspect of the project. |
| | `sourceFiles` | `List<String>` | no | File paths whose insights contributed to this block. |
| `ProjectMemory` | `blocks` | `List<MemoryBlock>` | no | All memory blocks for the project. |
| | `systemPrompt` | `String` | no | Rewritten agent system prompt (2–3 sentences). |
| | `builtAt` | `Instant` | no | When `MemoryWriterAgent` produced this memory. |
| `EditRequest` | `projectId` | `String` | no | Project to edit. |
| | `instruction` | `String` | no | Free-text edit instruction from the user. |
| | `requestedBy` | `String` | no | UI identifier of the user. |
| `FileEdit` | `filePath` | `String` | no | Absolute path under `/workspace/`. |
| | `kind` | `EditKind` | no | INSERT / REPLACE / DELETE. |
| | `patch` | `String` | no | Unified-diff-like text (empty for DELETE). |
| `PatchPlan` | `edits` | `List<FileEdit>` | no | Ordered list of 1–4 file edits. |
| | `rationale` | `String` | no | 1–2 sentence justification. |
| `PatchResult` | `filePath` | `String` | no | Path that was (or was not) edited. |
| | `ok` | `boolean` | no | True if the edit applied cleanly. |
| | `diff` | `Optional<String>` | yes | Unified-diff of the applied change; present when `ok=true`. |
| | `errorReason` | `Optional<String>` | yes | Why the edit failed; present when `ok=false`. |
| `TestResult` | `passed` | `boolean` | no | True if all tests passed. |
| | `total` | `int` | no | Total test count. |
| | `failed` | `int` | no | Failed test count. |
| | `output` | `Optional<String>` | yes | Test runner output snippet; present when `passed=false`. |
| `PatchEntry` | `attempt` | `int` | no | 1-based index for this edit in the plan. |
| | `edit` | `FileEdit` | no | The edit that was (or was not) applied. |
| | `verdict` | `PatchVerdict` | no | APPLIED / BLOCKED_BY_GUARDRAIL / FAILED / TESTS_FAILED. |
| | `result` | `Optional<PatchResult>` | yes | Present when the executor ran (even if `ok=false`). |
| | `testResult` | `Optional<TestResult>` | yes | Present after the test-gate step ran. |
| | `blocker` | `Optional<String>` | yes | Guardrail or executor failure reason when applicable. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `Project` (entity state) | `projectId` | `String` | no | Unique project id. |
| | `projectPath` | `String` | no | Root path supplied on init. |
| | `status` | `ProjectStatus` | no | RESEARCHING / READY / FAILED. |
| | `plan` | `Optional<ResearchPlan>` | yes | Populated after `ResearchPlanReady`. |
| | `memory` | `Optional<ProjectMemory>` | yes | Populated after `MemoryBuilt`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `ProjectFailed`. |
| | `createdAt` | `Instant` | no | When `ProjectCreated` was emitted. |
| | `readyAt` | `Optional<Instant>` | yes | When `SystemPromptRewritten` was emitted. |
| `EditSession` (entity state) | `sessionId` | `String` | no | Unique session id. |
| | `projectId` | `String` | no | Project this session belongs to. |
| | `instruction` | `String` | no | Edit instruction. |
| | `status` | `SessionStatus` | no | See enum. |
| | `patchPlan` | `Optional<PatchPlan>` | yes | Populated after `PatchPlanReady`. |
| | `patchLog` | `List<PatchEntry>` | no | Append-only log of all patch attempts. |
| | `failureReason` | `Optional<String>` | yes | Populated on `SessionFailed` / `SessionTimedOut`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `SessionHalted`. |
| | `createdAt` | `Instant` | no | When `SessionCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the session reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |

## Enums

- `EditKind` → `INSERT`, `REPLACE`, `DELETE`.
- `PatchVerdict` → `APPLIED`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `TESTS_FAILED`.
- `ProjectStatus` → `RESEARCHING`, `READY`, `FAILED`.
- `SessionStatus` → `PLANNING`, `APPLYING`, `COMPLETED`, `TESTS_FAILED`, `FAILED`, `HALTED`, `TIMED_OUT`.

## Events (`ProjectEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ProjectCreated` | `projectId, projectPath, createdAt` | → RESEARCHING |
| `ResearchPlanReady` | `plan: ResearchPlan` | no status change; sets `project.plan`. |
| `MemoryBuilt` | `memory: ProjectMemory` | no status change; sets `project.memory`. |
| `SystemPromptRewritten` | `systemPrompt, rewrittenAt` | → READY, `readyAt = now` |
| `ProjectFailed` | `failureReason` | → FAILED |
| `EditSessionRequested` | `sessionId, instruction, requestedBy, requestedAt` | no status change; triggers `SessionRequestConsumer`. |

## Events (`EditSessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionCreated` | `sessionId, projectId, instruction, createdAt` | → PLANNING |
| `PatchPlanReady` | `patchPlan: PatchPlan` | → APPLYING |
| `PatchBlocked` | `attempt, edit, blocker` | no status change; appends `PatchEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `PatchApplied` | `entry: PatchEntry` | no status change; appends to `patchLog`. |
| `TestsPassed` | `entry: PatchEntry (updated with testResult)` | no status change; updates last `patchLog` entry. |
| `TestsFailed` | `testResult, failureReason` | → TESTS_FAILED, `finishedAt = now` |
| `SessionCompleted` | `finishedAt` | → COMPLETED, `finishedAt = now` |
| `SessionFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `SessionHalted` | `haltReason` | → HALTED, `finishedAt = now` |
| `SessionTimedOut` | `failureReason` | → TIMED_OUT, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## View rows

`ProjectRow` mirrors `Project` but replaces the full `memory.blocks` list with `memoryBlockCount: int` and `systemPromptPreview: String` (first 200 chars). Every nullable field is declared `Optional<T>` (Lesson 6). The UI fetches the full project by id on expand via `GET /api/projects/{id}`.

`SessionRow` mirrors `EditSession` but truncates `patchLog` to the last 3 entries plus `truncatedFromTotal: int`. Every nullable field is declared `Optional<T>`. The UI fetches the full session by id on expand via `GET /api/sessions/{id}`.
