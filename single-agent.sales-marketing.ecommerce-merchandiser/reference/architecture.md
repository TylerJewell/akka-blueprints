# Architecture — merchandiser

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ProposalEndpoint` accepts a submission and writes an `ObjectiveSubmitted` event onto `ProposalEntity`. The `CatalogReader` Consumer subscribes, builds a `CatalogContext` snapshot from the catalog seed data filtered to the submitted `ProductScope`, and writes the context back via `attachContext`. The same Consumer then starts a `ProposalWorkflow` instance. The workflow's `generateStep` calls `MerchandiserAgent` — the single AutonomousAgent — with the merchant's objective as `TaskDef.instructions(...)` and the catalog snapshot as a `TaskDef.attachment(...)`. The agent's `before-tool-call` guardrail (`StorefrontGuardrail`) intercepts every tool call the agent attempts during generation: read-class calls pass through; write-class calls are rejected with a structured error that instructs the agent to surface the intended change as a `ChangeRecommendation` instead. Once the agent returns a `MerchandisingProposal`, the workflow writes `ProposalGenerated` and halts at `awaitApprovalStep`. The merchant approves or rejects via `ProposalEndpoint`. On approval, `publishStep` executes the write-class tool calls for real — this is the only path where write tools execute. `ProposalView` projects every entity event into a read-model row; `ProposalEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The approval gate (`awaitApprovalStep`) is a workflow pause on human input, not an LLM decision. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Three distinct pause points:

1. The `CatalogReader` subscription lag between `ObjectiveSubmitted` and `CatalogContextLoaded` — sub-second in normal operation.
2. The `awaitContextStep` polling loop inside the workflow — polls `ProposalEntity` every 1 s up to its 15 s timeout.
3. The `awaitApprovalStep` polling loop — polls every 5 s up to the 72-hour approval timeout. This is the human-in-the-loop boundary: the workflow does not advance until a merchant explicitly acts.

The agent call itself is bounded by `generateStep`'s 90 s timeout, which accommodates multi-turn tool calls (the agent may call `fetchProduct` or `searchCatalog` before finalising the proposal).

## State machine

Nine states. The interesting paths:

- The happy path is `SUBMITTED → CONTEXT_LOADED → GENERATING → PENDING_APPROVAL → PUBLISHING → PUBLISHED`.
- Two terminal-rejection paths: `PENDING_APPROVAL → REJECTED` (merchant rejects) and `PENDING_APPROVAL → EXPIRED` (72-hour timeout fires).
- Two failure paths land in `FAILED`: a context error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `GENERATING`.
- There is no auto-approve path. The system never advances from `PENDING_APPROVAL` to `PUBLISHING` without an explicit `POST /api/proposals/{id}/approve` call.

## Entity model

`ProposalEntity` is the source of truth. It emits nine event types. `ProposalView` projects every event into a row used by the UI. `CatalogReader` subscribes to entity events to build the catalog context. `ProposalWorkflow` both reads (`getProposal`) and writes (`markGenerating`, `recordProposal`, `markPublishing`, `markPublished`, `expire`, `fail`) on the entity. The relationship between `MerchandiserAgent` and `MerchandisingProposal` is "returns" — the agent's task result is the proposal record.

## Governance flow

For any change that lands in a `PUBLISHED` entity, it passed through:

1. **CatalogReader** — the agent only sees a snapshot filtered to the submitted scope; it does not have direct access to the full storefront.
2. **MerchandiserAgent** — one model call, structured `MerchandisingProposal` output. No live writes during this step.
3. **before-tool-call guardrail** — any write-class tool call the agent attempts during generation is blocked before it executes. The agent cannot bypass this by trying a different write tool name.
4. **Merchant approval gate** — `awaitApprovalStep` holds the proposal until a human explicitly approves it. The approval decision, approver identity, timestamp, and any note are recorded on the entity for audit.

Each step is independent. Removing the guardrail opens the generation phase to live writes. Removing the approval gate publishes every proposal without human review. Neither gap is silently covered by the other.
