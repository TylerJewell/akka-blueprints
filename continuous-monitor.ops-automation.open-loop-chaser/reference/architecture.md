# Architecture — open-loop-chaser

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`ActionItemPoller` is the heartbeat — a TimedAction that ticks every 20 s and writes simulated `SourceEventReceived` events into `SourceEventQueue` (event-sourced for audit). A `PiiSanitizer` Consumer subscribes to that queue, redacts the source text and author, and emits `ActionItemSanitized` on each source event before starting an `ExtractionWorkflow`. The workflow calls `ActionItemExtractorAgent` (typed Agent), receives a list of extracted items, and creates one `ActionItemEntity` per item.

`StaleItemScanner` runs alongside as a second TimedAction, ticking every 10 minutes. It queries `ActionItemView` for items in PENDING or NUDGED status whose last nudge (or detection time) exceeds the staleness threshold. For each stale item it starts a `NudgeWorkflow`, which calls `NudgeComposerAgent`, passes through the before-tool-call guardrail, and dispatches the nudge (simulated).

## Interaction sequence

The sequence traces the combined happy path for a new source event arriving and a subsequent nudge being dispatched. There are two notable coordination points: the `ExtractionWorkflow` fan-out that creates multiple `ActionItemEntity` instances from a single source event, and the `NudgeWorkflow` guardrail gate that halts dispatch when the item's `ownerConfirmation` is absent.

The `guardStep` inside `NudgeWorkflow` is the system's primary safety mechanism against automated messages reaching unverified recipients.

## State machine

Ten states. The interesting branching:

- After PENDING, the item waits until `StaleItemScanner` marks it STALE. An operator may close it directly from PENDING without a nudge occurring.
- STALE transitions to NUDGED after a successful nudge dispatch. NUDGED can cycle back to STALE if the item remains unresolved past the next staleness window — supporting multi-nudge scenarios.
- SNOOZED is a temporary suppression: `SnoozeLapsed` brings the item back to PENDING so it re-enters the normal staleness check cycle.
- FAILED is a terminal state reached either by an extraction error or by the guardrail blocking a nudge when no confirmed owner is present.

## Entity model

`ActionItemEntity` is the source of truth; it emits ten distinct event types covering the full lifecycle. `SourceEventQueue` is the upstream audit log — it retains raw (pre-sanitization) source events and only `PiiSanitizer` subscribes to it. The projection to `ActionItemView` omits the raw source text; the audit log keeps it.

## Defence-in-depth governance flow

For any nudge that is dispatched, the item passed through:
1. **PII sanitizer** — never let the model see personal identifiers from the source content.
2. **ActionItemExtractorAgent** — typed extractor; returns an empty list rather than fabricating items.
3. **Human owner confirmation** — an operator must explicitly confirm the recipient before nudge composition begins.
4. **before-tool-call guardrail** — even if the workflow reaches `sendNudge`, the call is blocked unless `ownerConfirmation.isPresent()` is verified at call time.

Each layer is independent. Removing any one of them does not silently widen the system; it explicitly opens a risk surface that the operator dashboard is likely to surface through accumulating FAILED or unowned items.
