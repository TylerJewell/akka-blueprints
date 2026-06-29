# Architecture — akka-llm-pipeline-workflow

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on a `ContentWorkflow` that drives two typed LLM activities — no autonomous agent. `PostEndpoint` accepts a `{topic, wordTarget}` POST, writes `PostCreated` onto `PostEntity`, and starts `ContentWorkflow` keyed by `"workflow-" + postId`. The workflow's first step (`outlineStep`) emits `OutlineStarted`, then calls `OutlineActivity` — a single typed LLM call that returns a `PostOutline`. Before the workflow can advance, `SchemaGuardrail` intercepts the response at the `after-llm-response` hook and validates the outline's structural constraints. A valid outline causes the workflow to write `OutlineProduced` onto the entity and advance to `draftStep`. An invalid outline causes the workflow to write `ValidationFailed` and transition to `FAILED` via `failStep`.

`draftStep` reads the recorded `PostOutline` from the entity and passes it as the sole context to `DraftActivity` — a second single typed LLM call that returns a `BlogPost`. `EditorialGuardrail` intercepts the draft at the `before-agent-response` hook. A draft that passes all three editorial rules causes the workflow to write `PostApproved` and transition to `READY`. A draft that fails any rule causes the workflow to write `PostFlagged` and transition to `FLAGGED`.

`PostView` projects every entity event into a read-model row. `PostEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly two LLM-calling components (`OutlineActivity` and `DraftActivity`). Both guardrails are rule-based and deterministic — neither makes an LLM call.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The activity boundary IS the typed handoff. Between `outlineStep` and `draftStep`, the workflow writes `OutlineProduced` onto the entity. `draftStep` then reads `outline` from the entity to build the draft LLM call's context. The raw `topic` string is never passed to `DraftActivity` — the `PostOutline` is the only information path between the two LLM calls.
2. Each guardrail runs synchronously in-process, before the workflow commits the next state transition. `SchemaGuardrail` runs before `OutlineProduced` is written; `EditorialGuardrail` runs before `PostApproved` or `PostFlagged` is written. This ordering means the entity event log is authoritative: an `OutlineProduced` event guarantees the outline passed schema validation; a `PostApproved` event guarantees the draft passed editorial review.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → OUTLINING → OUTLINED → DRAFTING → DRAFTED → REVIEWING → READY`.
- `FAILED` is reached from `OUTLINING` when `SchemaGuardrail` rejects the outline (`ValidationFailed` event) or from any step when an unrecoverable error occurs (`PostFailed` event).
- `FLAGGED` is reached from `REVIEWING` when `EditorialGuardrail` catches a policy violation (`PostFlagged` event). The draft exists on the entity and is visible in the UI — the flag is a hold, not a discard.
- `READY` and `FLAGGED` and `FAILED` are all terminal. None has a resume path within the same workflow run; the user resubmits a new topic to get a fresh attempt.

## Entity model

`PostEntity` is the source of truth. It emits ten event types covering creation, each phase's start and completion, the two terminal happy-path states, the failure, and the validation-failure side-event. `PostView` projects every event into a row used by the UI. `ContentWorkflow` both reads (`getPost`) and writes (all commands) on the entity.

The relationship between each activity and its typed result is "returns" — `OutlineActivity` returns `PostOutline`; `DraftActivity` returns `BlogPost`. These typed results become the payloads of `OutlineProduced` and `DraftProduced` events respectively.

## Defence-in-depth governance flow

For any post that lands in the entity log, the topic passed through:

1. **Schema guardrail (after-llm-response)** — the outline LLM response is validated before it is written onto the entity. A malformed outline never reaches the draft activity.
2. **Draft activity (one LLM call)** — the outline is the sole context. The draft is bounded by the structure and word target the outline declares.
3. **Editorial guardrail (before-agent-response)** — the finished draft is checked for word-count, prohibited content, and section coverage before the post is committed as `READY`.

Each check is independent. The schema guardrail does not check editorial policy; the editorial guardrail does not re-validate the outline's structure. Removing one of them opens an explicit gap the other does not silently cover.
