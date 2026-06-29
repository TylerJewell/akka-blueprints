# Architecture — social-amplifier

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `AmplificationEndpoint` accepts a `{sourceUrl, targetPlatforms}` POST, writes `AmplificationCreated` onto `AmplificationEntity`, and starts `AmplificationWorkflow` keyed by `"amp-" + amplificationId`. The workflow's first step (`parseStep`) emits `ParseStarted`, then calls `AmplifierAgent` with `TaskDef.taskType(PARSE_ARTICLE)` and a `phase = PARSE` metadata tag. The agent invokes `ParseTools.fetchArticle` and `ParseTools.extractKeyMessages`; once it returns a `ParsedArticle`, the workflow writes `ArticleParsed` onto the entity and advances to `draftStep`.

`draftStep` emits `DraftStarted`, calls the agent with the DRAFT task, and routes the agent's `DraftSet` response through `BrandPolicyGuardrail`'s response hook before the workflow can record `DraftsProduced`. Any draft that fails brand-policy rules triggers a `BrandCheckFailed` event and the agent retries within its 4-iteration budget. Once a compliant `DraftSet` is accepted, `draftStep` writes `DraftsProduced` and advances to `publishStep`.

`publishStep` emits `PublishStarted`, calls the agent with the PUBLISH task, and filters every publish tool call through `BrandPolicyGuardrail`'s tool-call hook. Publish tools are blocked until `status ∈ {DRAFTED, PUBLISHING}` AND `draftSet.isPresent()` — the second line of defence against premature write side-effects. Once the agent returns a `PublishedSet`, the workflow writes `PostsPublished` and runs `evalStep`, which calls `BrandPolicyScorer` deterministically and writes `BrandAuditCompleted`. `AmplificationView` projects every event into a read-model row; `AmplificationEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `BrandPolicyScorer` is a deterministic rule-based scorer with no LLM call.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `draftStep` and `publishStep`, the workflow writes `DraftsProduced` onto the entity. The next step then reads `draftSet` from the entity to build the PUBLISH task's instruction context. The agent never sees draft-phase context inside the publish task's conversation; the typed handoff is the only path information travels.
2. Every DRAFT response is filtered through `BrandPolicyGuardrail`'s response hook, and every PUBLISH tool call is filtered through the same guardrail's tool-call hook. The two hooks are independent: removing one does not cause the other to silently compensate.

The agent calls themselves are bounded by per-step timeouts (60 s on parse / draft / publish). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Eight states. The interesting paths:

- The happy path walks `CREATED → PARSING → PARSED → DRAFTING → DRAFTED → PUBLISHING → PUBLISHED → (BrandAuditCompleted terminal)`.
- Three failure transitions land in `FAILED`: an agent error or exhausted iteration budget during `PARSING`, `DRAFTING`, or `PUBLISHING`.
- `BrandCheckFailed` and `PublishToolBlocked` are side-events recorded for audit; they do not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `SCHEDULED` state. The blueprint publishes immediately upon DRAFT phase completion; a deployer that needs human approval before publishing would insert an approval gate between `draftStep` and `publishStep`.

## Entity model

`AmplificationEntity` is the source of truth. It emits eleven event types — three lifecycle starts, three lifecycle completions, the brand audit, the two guardrail audit events, the failure, and the initial creation. `AmplificationView` projects every event into a row used by the UI. `AmplificationWorkflow` both reads (`getAmplification`) and writes all command methods on the entity. The relationship between `AmplifierAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any amplification run that reaches `PUBLISHED`, the content passed through:

1. **Brand-policy response guardrail** — every DRAFT task response is filtered. A `DraftSet` with a character-limit violation, a forbidden word, a missing required hashtag, or an informal-tone marker on a professional platform is rejected before the workflow records it; a `BrandCheckFailed` event records the violation for audit.
2. **AmplifierAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **Publish-intent tool-call guardrail** — every PUBLISH task tool call is filtered. A publish tool called before `DraftsProduced` is recorded is rejected; a `PublishToolBlocked` event records the blocked call.
4. **On-decision brand evaluator** — every published run gets a 1–5 brand-audit score. Receipt coverage, no stub-error receipts, low retry counts, and within-limit character counts are each worth one point on a base of 1.

Each check is independent. The response guardrail does not check publish-intent; the tool-call guardrail does not check brand-policy content. Removing one opens an explicit gap the others do not silently cover.
