# Data model — guardrails-pattern

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PromptRequest` | `interactionId` | `String` | no | UUID minted by `InteractionEndpoint`. |
| | `promptText` | `String` | no | Raw user-submitted text. |
| | `policyProfile` | `PolicyProfile` | no | Enum: GENERAL / CONSERVATIVE / STRICT. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `GuardrailOutcome` | `passed` | `boolean` | no | True if the guardrail allowed the input/output. |
| | `ruleId` | `String` | yes | Matched pattern identifier; null when passed. |
| | `reason` | `String` | yes | Human-readable explanation; null when passed. |
| | `checkedAt` | `Instant` | no | When the check completed. |
| `AgentReply` | `category` | `ReplyCategory` | no | Enum value: FACTUAL / CREATIVE / EXTRACTION / REFUSAL / OTHER. |
| | `text` | `String` | no | Reply body. |
| | `confidence` | `double` | no | 0.0–1.0 confidence estimate. |
| | `citations` | `List<String>` | no | Source labels; empty list if none. |
| | `repliedAt` | `Instant` | no | When the agent returned the reply. |
| `Interaction` (entity state) | `interactionId` | `String` | no | — |
| | `request` | `Optional<PromptRequest>` | yes | Populated after `PromptSubmitted`. |
| | `inputGuardResult` | `Optional<GuardrailOutcome>` | yes | Populated after `PromptGuardChecked` or `PromptBlocked`. |
| | `reply` | `Optional<AgentReply>` | yes | Populated after `ReplyRecorded`. |
| | `outputGuardResult` | `Optional<GuardrailOutcome>` | yes | Populated after `ReplyGuardChecked`. |
| | `status` | `InteractionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PromptSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Interaction` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`PolicyProfile`: `GENERAL`, `CONSERVATIVE`, `STRICT`.
`ReplyCategory`: `FACTUAL`, `CREATIVE`, `EXTRACTION`, `REFUSAL`, `OTHER`.
`InteractionStatus`: `SUBMITTED`, `PROMPT_CHECKED`, `BLOCKED`, `REPLYING`, `REPLY_RECORDED`, `FAILED`.

## Events (`InteractionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PromptSubmitted` | `request` | → SUBMITTED |
| `PromptGuardChecked` | `outcome` (passed=true) | → PROMPT_CHECKED |
| `PromptBlocked` | `ruleId: String, reason: String` | → BLOCKED (terminal) |
| `ReplyingStarted` | — | → REPLYING |
| `ReplyRecorded` | `reply` | → REPLY_RECORDED |
| `ReplyGuardChecked` | `outcome` | stays in REPLY_RECORDED (audit event) |
| `InteractionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Interaction.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`InteractionRow` mirrors `Interaction`. The full `promptText` is included — unlike the docreview blueprint there is no raw/sanitized split, so the view row is the complete row. Deployers who need to omit prompt text from the view for data-minimization reasons should filter it in the table updater.

The view declares ONE query: `getAllInteractions: SELECT * AS interactions FROM interaction_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`AssistantTasks.java`)

```java
public final class AssistantTasks {
  public static final Task<AgentReply> GENERATE_REPLY = Task
      .name("Generate reply")
      .description("Read the user prompt and produce a structured AgentReply with category, confidence, text, and citations")
      .resultConformsTo(AgentReply.class);

  private AssistantTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Guardrail hook registration

Both guardrails are registered on `AssistantAgent.definition()`:

```java
AgentDefinition.of(this)
    .instructions(loadSystemPrompt())
    .capability(TaskAcceptance.of(AssistantTasks.GENERATE_REPLY).maxIterationsPerTask(3))
    .guardrail(PromptGuardrail.class, GuardrailHook.BEFORE_LLM_CALL)
    .guardrail(ReplyGuardrail.class, GuardrailHook.BEFORE_AGENT_RESPONSE);
```

`PromptGuardrail.BEFORE_LLM_CALL` and `ReplyGuardrail.BEFORE_AGENT_RESPONSE` reflect the canonical registration pattern from the Akka agent docs. Verify the exact method signature against `akka-context/sdk/ai-coding-assistant-guidelines.html.md` at generation time.
