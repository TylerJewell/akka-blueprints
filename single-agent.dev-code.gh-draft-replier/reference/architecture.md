# Architecture — gitty

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `DraftEndpoint` accepts a queue request and writes a `ThreadQueued` event onto `DraftEntity`. The `ThreadFetcher` Consumer subscribes to that event, calls the GitHub REST API to pull the issue or PR thread, and writes the result back via `attachThread`. The same Consumer then starts a `DraftWorkflow` instance. The workflow's `draftStep` calls `ReplyDrafterAgent` — the single AutonomousAgent — with the maintainer's context note as `TaskDef.instructions(...)` and the serialised thread JSON as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`ToneGuardrail`) validates each candidate draft. Once a draft passes, the workflow writes `DraftProduced` and enters `pendingReviewStep`, which waits indefinitely for the maintainer to act. `DraftEndpoint` routes the maintainer's approve or discard action back into the workflow. `DraftView` projects every entity event into a read-model row; `DraftEndpoint` serves the read model over REST and SSE.

The graph has no second agent. `ThreadFetcher` is a Consumer — it calls the GitHub API but makes no LLM call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). There are three distinct wait points:

1. The `ThreadFetcher` subscription lag between `ThreadQueued` and `ThreadFetched` — typically sub-second for an in-process call; up to a few seconds if the GitHub API is slow.
2. The `awaitThreadStep` polling loop inside the workflow — polls `DraftEntity` every 1 s up to its 20 s timeout, advancing as soon as `draft.thread().isPresent()` is true.
3. `pendingReviewStep` — an indefinite wait with no timeout, held until the maintainer acts via the UI.

The agent call inside `draftStep` is bounded by a 60 s timeout. A guardrail rejection sends a structured error back to the agent loop, which retries within its 3-iteration budget.

## State machine

Seven states, including two terminal happy states and one terminal failure state. The interesting paths:

- The full happy path is `QUEUED → THREAD_FETCHED → DRAFTING → PENDING_REVIEW → APPROVED` (or `→ DISCARDED`).
- Two failure transitions land in `FAILED`: a fetch error during `QUEUED`, and an agent error (or guardrail-exhaustion) during `DRAFTING`. A `FAILED` draft's prior data is preserved on the entity; the UI shows the partial state so the maintainer knows what happened.
- `APPROVED` and `DISCARDED` are both terminal happy states — the maintainer acted. The distinction is recorded in the entity for audit, but neither posts to GitHub.
- There is no automatic re-queue. If the maintainer wants a fresh draft for the same thread, they submit a new request via the UI.

## Entity model

`DraftEntity` is the source of truth. It emits seven event types. `DraftView` projects every event into a row used by the UI list. `ThreadFetcher` subscribes to entity events to trigger the GitHub fetch. `DraftWorkflow` both reads (`getDraft`) and writes (`markDrafting`, `recordDraft`, `approveDraft`, `discardDraft`, `fail`) on the entity. `ReplyDrafterAgent` and `DraftReply` have a "returns" relationship — the agent's task result is the draft record.

## Governance flow

For any draft that moves through the system, two independent checks must pass before text reaches the maintainer's clipboard:

1. **ToneGuardrail** — the model's candidate response is validated inside the agent loop. A non-compliant draft is rejected before it leaves `ReplyDrafterAgent`. The entity log never contains a non-compliant draft text.
2. **HITL gate** — every compliant draft waits in `PENDING_REVIEW`. The maintainer reads it, optionally edits it, and explicitly acts. The system records the decision but does not post to GitHub; that step remains the maintainer's responsibility.

The two checks are independent: removing the guardrail would let non-compliant drafts reach the HITL queue; removing the HITL gate would allow compliant drafts to be posted without human review. Both are necessary.
