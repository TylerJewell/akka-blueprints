# Architecture — gitwiki

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `WikiEndpoint` accepts an update submission and writes an `UpdateSubmitted` event onto `PageEntity`, then starts a `WikiUpdateWorkflow` instance. The workflow's `configCheckStep` calls `StartupConfigGate.isHealthy()` to confirm the GitHub credential is present before doing any real work. The `editStep` then calls `WikiEditorAgent` — the single AutonomousAgent — with the requested changes as `TaskDef.instructions(...)` and the current page body as a `TaskDef.attachment(...)`. The agent's `before-tool-call` guardrail (`GitPushGuardrail`) runs on any push tool call the agent issues. Once a `CommitDraft` is returned, the workflow writes `CommitReady` on the entity. The `GitPushConsumer` subscribes to `CommitReady` events, calls `GitPushGuardrail` a second time as an explicit pre-push check, executes the GitHub API push, and writes the result back via `PageEntity.recordPushResult`. `PageView` projects every entity event into a read-model row; `WikiEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `GitPushGuardrail` and `StartupConfigGate` are plain Java classes. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two points where the system waits on something:

1. The `configCheckStep` startup probe — a single HTTP call to GitHub, sub-second when the token is valid.
2. The `awaitPushStep` polling loop inside the workflow — polls `PageEntity` every 1 s up to its 30 s timeout, advancing as soon as the update reaches a terminal state (`PUSHED`, `PUSH_REJECTED`, `CONFLICT`, or `FAILED`).

The agent call is bounded by `editStep`'s 60 s timeout. The GitHub push itself is a synchronous HTTP call inside `GitPushConsumer`; the consumer's event processing timeout bounds it.

## State machine

Eight states. The interesting paths:

- The happy path is `SUBMITTED → EDITING → COMMIT_READY → PUSH_IN_PROGRESS → PUSHED`.
- A guardrail rejection during the push lands in `PUSH_REJECTED` — a terminal state. The page content from the agent is still on the entity; a user can resubmit the update after correcting the branch configuration.
- A non-fast-forward GitHub response lands in `CONFLICT` — also terminal. The conflicting commit SHA is recorded for the user to resolve manually via `git rebase`.
- Two failure transitions land in `FAILED`: a config gate failure during `SUBMITTED`, and an agent error during `EDITING`. Prior data is preserved on the entity.
- There is no `APPROVED` state. The update is written directly to the target branch; a human reviewer acts via GitHub's branch-protection or PR flow, outside this system.

## Entity model

`PageEntity` is the source of truth. It emits seven event types. `PageView` projects every event into a row used by the UI. `GitPushConsumer` subscribes to entity events to execute the push. `WikiUpdateWorkflow` both reads (`getUpdate`) and writes (`markEditing`, `markCommitReady`, `markPushStarted`, `fail`) on the entity. `GitPushConsumer` writes the final result via `recordPushResult`. The relationship between `WikiEditorAgent` and `CommitDraft` is "returns" — the agent's task result is the draft record.

## Defence-in-depth governance flow

For any commit that reaches the remote repository, the update passed through:

1. **Configuration gate** — the service refused to start without a valid GitHub token and explicit branch config.
2. **WikiEditorAgent** — one model call, one structured output.
3. **before-tool-call guardrail** — target ref, author identity, and page namespace are checked before any push tool call executes inside the agent loop.
4. **GitPushConsumer guardrail check** — the same three properties are re-checked at the Consumer layer before the GitHub API call, providing defence-in-depth for the case where the agent loop's tool-call pathway is bypassed.

Each layer is independent. Removing one opens an explicit gap the others do not silently cover.
