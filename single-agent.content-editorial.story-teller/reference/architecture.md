# Architecture — story-teller

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `StoryEndpoint` accepts a submission and writes a `StoryRequested` event onto `StoryEntity`. The `PromptEnricher` Consumer subscribes, runs a content-safety check plus metadata tagging, and writes the enriched prompt back via `attachEnriched` — or calls `block(reason)` if the prompt fails the safety check. When enrichment succeeds, the same Consumer starts a `StoryWorkflow` instance. The workflow's `generationStep` calls `StoryTellerAgent` — the single AutonomousAgent — with the style constraints as `TaskDef.instructions(...)` and the enriched prompt as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`StoryGuardrail`) validates each candidate response. Once a story passes, the workflow writes `StoryRecorded` and runs `QualityScorer` in `qualityStep`. The score lands as `QualityScored`. `StoryView` projects every entity event into a read-model row; `StoryEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The on-decision evaluator (`QualityScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct moments where the system waits on something:

1. The `PromptEnricher` subscription lag between `StoryRequested` and `PromptEnriched` — sub-second in normal operation; the check is keyword-based with no external call.
2. The `awaitEnrichedStep` polling loop inside the workflow — polls `StoryEntity` every 1 s up to its 15 s timeout, advancing as soon as `story.enriched().isPresent()` returns true (or terminating early if `status == BLOCKED`).

The agent call itself is bounded by `generationStep`'s 60 s timeout. The `qualityStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Seven states. The interesting paths:

- The happy path is `REQUESTED → ENRICHED → GENERATING → STORY_RECORDED → SCORED`.
- A safety-rejected prompt follows `REQUESTED → BLOCKED` — a terminal state reached without any agent call. The block reason is preserved on the entity for audit.
- Two failure transitions land in `FAILED`: an enricher error during `REQUESTED`, and an agent error (or guardrail-exhaustion) during `GENERATING`. A `FAILED` story's prior data is preserved on the entity.
- There is no moderation or approval state. The system assumes the content-safety filter and guardrail are the governance boundary; downstream publication decisions belong to the deployer.

## Entity model

`StoryEntity` is the source of truth. It emits seven event types. `StoryView` projects every event into a row used by the UI. `PromptEnricher` subscribes to entity events to compute the safe prompt and trigger the workflow. `StoryWorkflow` both reads (`getStory`) and writes (`markGenerating`, `recordStory`, `recordQuality`, `fail`) on the entity. The relationship between `StoryTellerAgent` and `GeneratedStory` is "returns" — the agent's task result is the story record.

## Defence-in-depth governance flow

For any story that lands in the entity log, the prompt passed through:

1. **Content-safety filter** — a keyword-based blocklist in `PromptEnricher` rejects prompts containing disallowed content before the agent is invoked; the raw prompt is preserved on the entity for audit.
2. **StoryTellerAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — empty bodies, genre mismatches, and missing author notes are caught before the response leaves the agent loop.
4. **On-decision quality evaluator** — every well-formed story still gets a 1–5 quality score so downstream consumers know which outputs are structurally thin.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
