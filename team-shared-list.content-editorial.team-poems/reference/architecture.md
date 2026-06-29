# Architecture — team-poems-multi-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`PoetryEndpoint` is the entry point. A submitted prompt is logged as a `PromptSubmitted` event on `PromptQueue` (event-sourced for audit). `PromptSubmissionConsumer` subscribes to that queue, creates a `PoemEntity`, and starts a `DirectionWorkflow`. The workflow runs the `PoetryDirector` agent to decompose the prompt into stanzas, then writes one `StanzaEntity` per assignment onto the board (each `OPEN`). Every stanza transition projects into `VerseBoardView` — the shared list.

The poet side runs in parallel and independently. `Bootstrap` starts one `PoetWorkflow` per poet id (`poet-1`, `poet-2`, `poet-3`). Each poet loop polls `VerseBoardView`, claims an eligible stanza on its `StanzaEntity`, runs the `PoetAgent`, runs the draft through the quality gate, and marks the stanza done or blocked. When a poet is blocked on tonal continuity it posts to a peer's `CoordinationMailbox`. `EditorialControl` (a key-value entity) holds the operator pause flag that both the poll loop and the content guardrail read. Two TimedActions sit alongside: `PromptSimulator` drips sample prompts; `StuckStanzaMonitor` releases stranded claims.

## Interaction sequence

The sequence diagram traces the happy path (J1): submit → plan → stanzas on the board → a poet claims, drafts, passes the quality gate, and the stanza goes `DONE`. The `Note over PoetAgent` block marks where the content guardrail vets tool calls. The poet loops are already polling before any stanza exists, so the only thing the direction step adds is rows on the board for them to find.

## State machine

`StanzaEntity` is the heart of the pattern. `OPEN` is the initial state. `claim` is atomic — only an `OPEN` stanza can be claimed, and the entity's single-writer guarantee means exactly one poet wins a contested stanza. From `CLAIMED` the poet moves to `DRAFTING`, submits a verse into `IN_REVIEW`, and the quality gate decides: `DONE` on pass, back to `DRAFTING` on a retryable failure, `BLOCKED` once retries are exhausted. A coordination request or a content guardrail refusal moves the stanza to `BLOCKED` directly. `StuckStanzaMonitor` returns a claimed-but-idle stanza to `OPEN` for liveness, and a coordination reply (or editor reopen) returns a blocked stanza to `OPEN`.

## Entity model

`StanzaEntity` is the source of truth for each unit of verse; every transition writes one of nine event types, and `VerseBoardView` is the only read-side projection — the poet loops never query the entity directly, they read the board. `PoemEntity` tracks the poem lifecycle and owns the set of stanza ids. `PromptQueue` is the audit log of submissions. `CoordinationMailbox` (one per poet) holds the cross-poet messages.

## Concurrency & timeouts

- Atomic claim through `StanzaEntity`'s single writer is the coordination primitive — no lock, no external queue.
- Per-step timeout: 90 s on the agent-calling steps (`DirectionWorkflow.directStep`, `PoetWorkflow.draftStep`).
- Idle poets are paused workflows with a 5 s resume timer, not busy loops.
- A stanza is eligible only when every `dependsOn` title resolves to a `DONE` stanza — checked client-side because the view exposes no enum-status filter.
- `StuckStanzaMonitor` releases a stanza claimed-but-idle for more than two minutes so a failed poet does not strand work.
- The quality gate (`QualityChecker`) is deterministic, so a given draft always yields the same verdict.
