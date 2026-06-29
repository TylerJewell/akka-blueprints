# Data model — guardrails-side-by-side

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PromptRequest` | `promptId` | `String` | no | UUID minted by `PromptEndpoint`. |
| | `promptText` | `String` | no | Raw user prompt text. |
| | `policyProfile` | `String` | no | `"standard"`, `"strict"`, or `"permissive"`. |
| | `submittedBy` | `String` | no | User identifier. |
| | `receivedAt` | `Instant` | no | When the endpoint received the request. |
| `ScreeningVerdict` | `result` | `ScreeningResult` | no | PASS or BLOCK. |
| | `reason` | `String` | no | One-sentence explanation. |
| | `decidedAt` | `Instant` | no | When `InputScreenerAgent` returned. |
| `PolicyResponse` | `reply` | `String` | no | Advisory text from `PolicyAgent`. |
| | `confidence` | `double` | no | 0.0 – 1.0. |
| | `respondedAt` | `Instant` | no | When `PolicyAgent` returned. |
| `ValidationVerdict` | `result` | `ValidationResult` | no | PASS or BLOCK. |
| | `reason` | `String` | no | One-sentence explanation. |
| | `triggeredRule` | `String` | yes | Rule name on BLOCK; null on PASS. |
| | `decidedAt` | `Instant` | no | When `OutputValidatorAgent` returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `ResponseEvaluator` finished. |
| `Prompt` (entity state) | `promptId` | `String` | no | — |
| | `request` | `Optional<PromptRequest>` | yes | Populated after `PromptReceived`. |
| | `screeningVerdict` | `Optional<ScreeningVerdict>` | yes | Populated after `PromptBlocked` or `AgentCallStarted`. |
| | `agentResponse` | `Optional<PolicyResponse>` | yes | Populated after `AgentResponded`. |
| | `validationVerdict` | `Optional<ValidationVerdict>` | yes | Populated after `ResponseBlocked` or `ResponseAccepted`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `PromptStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PromptReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `Prompt` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ScreeningResult`: `PASS`, `BLOCK`.
`ValidationResult`: `PASS`, `BLOCK`.
`PromptStatus`: `RECEIVED`, `SCREENING`, `AGENT_RUNNING`, `VALIDATING`, `RESPONDED`, `BLOCKED`, `RESPONSE_BLOCKED`, `FAILED`.

## Events (`PromptEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PromptReceived` | `request` | → RECEIVED |
| `ScreeningStarted` | — | → SCREENING |
| `PromptBlocked` | `screeningVerdict` | → BLOCKED (terminal) |
| `AgentCallStarted` | — | → AGENT_RUNNING |
| `AgentResponded` | `agentResponse` | → VALIDATING |
| `ValidationStarted` | — | stays VALIDATING |
| `ResponseBlocked` | `validationVerdict` | → RESPONSE_BLOCKED (terminal) |
| `ResponseAccepted` | `validationVerdict` | → RESPONDED (pending eval) |
| `EvaluationScored` | `eval` | stays RESPONDED (final terminal happy) |
| `PromptFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Prompt.initial("")` with all `Optional` fields as `Optional.empty()` and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`PromptRow` mirrors `Prompt`. The view declares ONE query: `getAllPrompts: SELECT * AS prompts FROM prompt_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`GuardrailTasks.java`)

```java
public final class GuardrailTasks {
  public static final Task<ScreeningVerdict> SCREEN_INPUT = Task
      .name("Screen input")
      .description("Evaluate the prompt for scope, safety, and injection risk")
      .resultConformsTo(ScreeningVerdict.class);

  public static final Task<PolicyResponse> ANSWER_PROMPT = Task
      .name("Answer prompt")
      .description("Produce a governance-risk advisory response to the prompt")
      .resultConformsTo(PolicyResponse.class);

  public static final Task<ValidationVerdict> VALIDATE_OUTPUT = Task
      .name("Validate output")
      .description("Check the agent reply against output policy rules")
      .resultConformsTo(ValidationVerdict.class);

  private GuardrailTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7). All three agents in this blueprint share one companion class.
