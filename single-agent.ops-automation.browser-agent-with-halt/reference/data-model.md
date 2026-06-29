# Data model — web-navigation-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskGoal` | `goalId` | `String` | no | UUID minted by `SessionEndpoint`; same as `sessionId`. |
| | `description` | `String` | no | User-supplied task goal text. |
| | `startingUrl` | `String` | no | URL where the browser session begins. |
| | `maxSteps` | `int` | no | Upper bound on action steps before auto-completion. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When `SessionEndpoint` received the request. |
| `BrowserAction` | `actionType` | `ActionType` | no | Enum value. |
| | `targetSelector` | `String` | no | CSS selector or descriptive label; empty for NAVIGATE/COMPLETE/REJECT_TASK. |
| | `targetUrl` | `String` | no | Destination URL; populated for NAVIGATE actions. |
| | `inputText` | `String` | no | Text to type; populated for TYPE actions. |
| | `rationale` | `String` | no | One-sentence explanation from the agent. |
| | `highStakes` | `boolean` | no | Agent-declared; guardrail may override to `true`. |
| | `decidedAt` | `Instant` | no | When the agent returned the action. |
| `ActionOutcome` | `actionId` | `String` | no | UUID minted per action step. |
| | `action` | `BrowserAction` | no | The action that was attempted. |
| | `executed` | `boolean` | no | `true` if browser executed; `false` if blocked. |
| | `blockedReason` | `Optional<String>` | yes | Guardrail block reason; empty when `executed=true`. |
| | `screenshotPath` | `Optional<String>` | yes | Path in snapshot store; populated after `ScreenshotCaptured`. |
| | `executedAt` | `Instant` | no | When the action resolved (executed or blocked). |
| `HitlDecision` | `decisionId` | `String` | no | UUID minted by `SessionEndpoint`. |
| | `actionId` | `String` | no | The `actionId` this decision covers. |
| | `outcome` | `HitlOutcome` | no | Enum value. |
| | `reviewedBy` | `String` | no | Human reviewer identifier. |
| | `decidedAt` | `Instant` | no | When the reviewer submitted the decision. |
| `SessionOutcome` | `success` | `boolean` | no | `true` if agent returned `COMPLETE` with goal achieved. |
| | `taskResult` | `String` | no | Agent's summary of what was found or done. |
| | `stepsUsed` | `int` | no | Number of action steps consumed. |
| | `completedAt` | `Instant` | no | When `completeStep` ran. |
| `Session` (entity state) | `sessionId` | `String` | no | — |
| | `goal` | `Optional<TaskGoal>` | yes | Populated after `SessionStarted`. |
| | `actionLog` | `List<ActionOutcome>` | no | Append-only; empty initially. |
| | `pendingAction` | `Optional<BrowserAction>` | yes | Populated during `AWAITING_APPROVAL`; cleared on `HitlResolved`. |
| | `lastHitlDecision` | `Optional<HitlDecision>` | yes | The most recent HITL decision. |
| | `outcome` | `Optional<SessionOutcome>` | yes | Populated after `SessionCompleted`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionStarted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `HaltState` (HaltController state) | `halted` | `boolean` | no | Current halt flag value. |
| | `setAt` | `Optional<Instant>` | yes | When the last `HaltSet` event was applied. |
| | `clearedAt` | `Optional<Instant>` | yes | When the last `HaltCleared` event was applied. |

Every nullable field on `Session` and `HaltState` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ActionType`: `CLICK`, `TYPE`, `SCROLL`, `NAVIGATE`, `SCREENSHOT`, `COMPLETE`, `REJECT_TASK`.
`HitlOutcome`: `APPROVED`, `REJECTED`.
`SessionStatus`: `STARTING`, `NAVIGATING`, `AWAITING_APPROVAL`, `COMPLETED`, `HALTED`, `FAILED`, `REJECTED`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionStarted` | `goal` | → STARTING |
| `ActionPlanned` | `action` | — (no status change; bookkeeping) |
| `ActionExecuted` | `outcome` | — (status unchanged; log grows) |
| `ScreenshotCaptured` | `screenshotPath` | — (updates latest screenshot on log entry) |
| `HitlRequested` | `pendingAction` | → AWAITING_APPROVAL |
| `HitlResolved` | `decision` | → NAVIGATING |
| `SessionCompleted` | `outcome` | → COMPLETED (terminal happy) |
| `SessionHalted` | — | → HALTED (terminal) |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |
| `SessionRejected` | `reason: String` | → REJECTED (terminal) |

## Events (`HaltController`)

| Event | Payload | Effect |
|---|---|---|
| `HaltSet` | `setAt: Instant` | `halted` = true |
| `HaltCleared` | `clearedAt: Instant` | `halted` = false |

`emptyState()` returns `Session.initial("")` with all `Optional` fields as `Optional.empty()`, `actionLog` as `Collections.emptyList()`, and `status = STARTING`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `Session` but replaces the full `actionLog` list with `actionCount: int` and `latestActionSummary: Optional<String>` (the most recent action's rationale and type as a short string) for the UI list. The full action log is fetched on demand via `GET /api/sessions/{id}`.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`NavigationTasks.java`)

```java
public final class NavigationTasks {
  public static final Task<BrowserAction> PLAN_NEXT_ACTION = Task
      .name("Plan next browser action")
      .description("Observe the current page screenshot and task goal; return the single next BrowserAction")
      .resultConformsTo(BrowserAction.class);

  private NavigationTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
