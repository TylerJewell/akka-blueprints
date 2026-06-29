# Architecture — feed-monitor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`FeedPoller` is the heartbeat — a TimedAction that ticks every 60 s and writes `ItemReceived` events into `FeedItemEntity`. Each `ItemReceived` event starts a `FeedWorkflow` instance, which runs two agent steps in sequence: `SummaryAgent` (typed, summarizes the raw content) and `ClassifierAgent` (typed, decides NOTIFY / SUPPRESS / REVIEW). On NOTIFY, the workflow emits `NotifyRequested`, which `SlackNotifier` Consumer picks up asynchronously. `SlackNotifier` runs the before-tool-call guardrail before dispatching the stub Slack post.

`EvalRunner` runs alongside as a second TimedAction, ticking every 30 minutes and scoring a sample of POSTED items via `EvalJudge`.

## Interaction sequence

The sequence traces the NOTIFY happy path. Two key asynchronous handoffs exist: (1) the workflow step calling `SummaryAgent` (blocking on the LLM call, up to 20 s); (2) the `NotifyRequested` event being picked up by `SlackNotifier` Consumer (asynchronous, sub-second under normal load). The guardrail check inside `SlackNotifier` is synchronous — it runs before the stub call and hard-rejects if the channel is absent from the allow-list.

## State machine

Eight active states plus terminal states. The key branches:

- After CLASSIFIED, the classification value branches: NOTIFY → NOTIFY_REQUESTED → (if guardrail passes) POSTED terminal, or → FAILED terminal if guardrail rejects. SUPPRESS → SUPPRESSED terminal. REVIEW → PENDING_REVIEW, which can be advanced by a human via `FeedEndpoint.override` (→ NOTIFY_REQUESTED) or `FeedEndpoint.suppress` (→ SUPPRESSED).
- There is no auto-timeout on PENDING_REVIEW. Items wait until a human acts.
- FAILED is a terminal state used for both guardrail rejections and agent step timeouts.

## Entity model

`FeedItemEntity` is the source of truth; it emits ten distinct event types covering the full lifecycle including eval. `SlackNotifier` Consumer subscribes to `NotifyRequested` events and writes back `ItemPosted` or `ItemFailed`. `FeedView` projects all events into a flat read-model row per item.

## Governance flow

For any item that reaches POSTED, it passed through:

1. **SummaryAgent** — condensed the raw content without invention.
2. **ClassifierAgent** — decided NOTIFY with explicit channel targeting.
3. **before-tool-call guardrail** — validated the channel against the allow-list before the Slack call.
4. **eval-periodic sampler** — scored the posted summary retroactively for quality monitoring.

Each layer is independent. The guardrail can catch misrouted channel values even if the classifier produced a valid classification. The eval sampler detects quality drift across the full output stream over time.
