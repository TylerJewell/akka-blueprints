# Data model — game-builder-team

Authoritative record definitions. Lifecycle fields that are null before their transition use `Optional<T>` (Lesson 6). Akka serializes `Optional<T>` as the raw value or `null` on the wire.

## Records

### `GameProject` (entity state + view row type)
| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Project / workflow id (UUID). |
| `idea` | `String` | no | The submitted game idea. |
| `status` | `GameStatus` | no | Lifecycle status. |
| `queuedAt` | `Optional<Instant>` | yes | When the build was queued. |
| `designedAt` | `Optional<Instant>` | yes | When the spec was recorded. |
| `spec` | `Optional<GameSpec>` | yes | The designer's output. |
| `codedAt` | `Optional<Instant>` | yes | When the code was recorded. |
| `code` | `Optional<GameCode>` | yes | The code writer's output. |
| `testedAt` | `Optional<Instant>` | yes | When the test gate last ran. |
| `testReport` | `Optional<TestReport>` | yes | Result of the test gate. |
| `testAttempts` | `Optional<Integer>` | yes | Number of code attempts so far. |
| `deliveredAt` | `Optional<Instant>` | yes | When the build was delivered. |
| `blockReason` | `Optional<String>` | yes | Why a build was blocked. |

`emptyState()` returns `GameProject.initial("", "")` with no `commandContext()` reference (Lesson 3).

### `GameSpec`
| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `title` | `String` | no | Game title. |
| `genre` | `String` | no | Game genre. |
| `mechanics` | `List<String>` | no | Two to four implementable mechanics. |
| `controls` | `String` | no | Player inputs. |
| `winCondition` | `String` | no | When the game ends or scores. |

### `GameCode`
| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `html` | `String` | no | Single self-contained HTML/JS game. |
| `entryPoint` | `String` | no | Function that starts the game. |
| `filesTouched` | `List<String>` | no | Logical files in the document. |

### `TestReport`
| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `passed` | `boolean` | no | Whether the test gate passed. |
| `checks` | `List<String>` | no | Checks that ran. |
| `failures` | `List<String>` | no | Checks that failed (empty when passed). |

## Status enum
```java
enum GameStatus { QUEUED, DESIGNING, CODING, TESTING, TEST_FAILED, DELIVERED, BLOCKED }
```

## Events (on `GameProjectEntity`)
| Event | Trigger |
|---|---|
| `BuildRequested` | Workflow starts; project enters `QUEUED`/`DESIGNING`. |
| `SpecDesigned` | Designer returns a `GameSpec`; sets `spec`, `designedAt`. |
| `CodeWritten` | Code writer returns a `GameCode`; sets `code`, `codedAt`. |
| `TestsPassed` | Test gate passes; sets `testReport`, `testedAt`. |
| `TestsFailed` | Test gate fails after the retry budget; sets `testReport`, `testAttempts`. |
| `GameDelivered` | Assemble step completes; sets `deliveredAt`, status `DELIVERED`. |
| `BuildBlocked` | Sandbox guardrail trips; sets `blockReason`, status `BLOCKED`. |

## Events (on `BuildRequestQueue`)
| Event | Trigger |
|---|---|
| `BuildEnqueued` | An idea is submitted via the endpoint or the simulator. |

## View row
`GameView` row type is `GameProject`. One query: `getAllProjects` — `SELECT * AS projects FROM game_view`. No `WHERE status` filter (Akka cannot auto-index enum columns, Lesson 2); status filtering is client-side in `GameEndpoint`.
