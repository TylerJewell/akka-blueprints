# Data model — skill-patterns-tutorial

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SkillRunRequest` | `runId` | `String` | no | UUID minted by `SkillRunEndpoint`. |
| | `pattern` | `SkillPattern` | no | One of the four pattern values. |
| | `patternInputJson` | `String` | no | JSON-encoded union; shape depends on pattern. |
| | `requestedBy` | `String` | no | User identifier from the request body. |
| | `requestedAt` | `Instant` | no | When the endpoint received the request. |
| `SkillResult` | `patternName` | `String` | no | Display name: "Inline", "File-based", "External", "Meta creator". |
| | `output` | `String` | no | The skill's produced text (or SkillDefinition JSON for META_CREATOR). |
| | `wiringNote` | `String` | no | One sentence describing how the skill was loaded. |
| | `completedAt` | `Instant` | no | When the agent task returned. |
| `SkillDefinition` | `skillId` | `String` | no | Kebab-case slug. |
| | `displayName` | `String` | no | Human-readable name. |
| | `systemPrompt` | `String` | no | Full prompt for the described skill. |
| | `createdBy` | `String` | no | Always "SkillDemoAgent" for agent-generated definitions. |
| | `createdAt` | `Instant` | no | When the META_CREATOR run completed. |
| `SkillRun` (entity state) | `runId` | `String` | no | — |
| | `request` | `Optional<SkillRunRequest>` | yes | Populated after `SkillRunRequested`. |
| | `result` | `Optional<SkillResult>` | yes | Populated after `SkillRunCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated after `SkillRunFailed`. |
| | `status` | `SkillRunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SkillRunRequested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `SkillRun` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Input union records (deserialized inside `SkillRunWorkflow` based on `pattern`)

| Record | Field | Type | Meaning |
|---|---|---|---|
| `InlineSkillInput` | `topic` | `String` | Topic to explain. |
| | `style` | `String` | One of: `concise`, `detailed`, `bullet-list`. |
| `FileBasedSkillInput` | `textToSummarize` | `String` | Input text for the summarizer skill. |
| `ExternalSkillInput` | `lookupKey` | `String` | Key sent to the tool stub. |
| `MetaCreatorInput` | `skillName` | `String` | Display name for the new skill. |
| | `skillDescription` | `String` | Free-text description for the agent to synthesize a prompt from. |

## Enums

`SkillPattern`: `INLINE`, `FILE_BASED`, `EXTERNAL`, `META_CREATOR`.

`SkillRunStatus`: `REQUESTED`, `RUNNING`, `COMPLETED`, `FAILED`.

## Events (`SkillRunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SkillRunRequested` | `request` | → REQUESTED |
| `SkillRunStarted` | — | → RUNNING |
| `SkillRunCompleted` | `result` | → COMPLETED (terminal happy) |
| `SkillRunFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `SkillRun.initial("")` with all `Optional` fields as `Optional.empty()` and `status = REQUESTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SkillRunRow` mirrors `SkillRun`. The view declares ONE query: `getAllRuns: SELECT * AS runs FROM skill_run_view`. No `WHERE status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`SkillDemoTasks.java`)

```java
public final class SkillDemoTasks {
  public static final Task<SkillResult> INVOKE_SKILL = Task
      .name("Invoke skill")
      .description("Run the requested skill pattern and return a SkillResult")
      .resultConformsTo(SkillResult.class);

  private SkillDemoTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Tool stub response shape

```java
record ToolLookupResponse(String key, String value, String source) {}
```

`source` is always `"in-process-stub"`. Five seeded keys: `alpha`, `beta`, `gamma`, `delta`, `epsilon`. Unknown keys return `value = "no-data-for-key:" + key` with HTTP 200.
