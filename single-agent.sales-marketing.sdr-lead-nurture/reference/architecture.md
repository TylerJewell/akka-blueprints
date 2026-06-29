# Architecture — sdr-lead-nurture

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `LeadEndpoint` accepts an inbound lead and writes a `LeadReceived` event onto `LeadEntity`. The `LeadSanitizer` Consumer subscribes, strips PII from the contact record, and writes the sanitized form back via `attachSanitized`. The same Consumer then starts a `LeadWorkflow` instance. The workflow's `engageStep` calls `SdrAgent` — the single AutonomousAgent — with the sanitized lead context as `TaskDef.instructions(...)`. Two guardrails sit on the agent: `ReplyGuardrail` checks every candidate outbound message before it leaves the agent loop; `BookingGuardrail` checks every write-action tool call before it is dispatched. Once an `AgentTurn` passes both guardrails, the workflow records it and, if a close-action tool call was included, dispatches the tool. At the end of engagement, `closeStep` runs `QualityScorer` — a deterministic rule-based scorer — and emits `QualityScoredEvent`. `LeadView` projects every entity event into a read-model row; `LeadEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one component that talks to a model: `SdrAgent`. `QualityScorer`, `ReplyGuardrail`, and `BookingGuardrail` are all in-process logic — no additional LLM calls. That is what makes this blueprint a faithful **single-agent** example.

## Interaction sequence

The sequence traces the happy path (J1): a lead arrives, is sanitized, the agent engages in a two-turn conversation, proposes a meeting, the guardrail accepts the slot, the meeting is booked, and the quality scorer runs. Two guardrail checkpoints stand out:

1. `ReplyGuardrail` runs on every outbound message candidate, before the reply is recorded on the entity. A brand-violating draft never reaches the lead.
2. `BookingGuardrail` runs on `BOOK_MEETING` before `CalendarStub.book` is called. An out-of-hours slot returns a structured error; the agent proposes a new slot within the same iteration.

The agent call itself is bounded by `engageStep`'s 60 s timeout. `closeStep` with `QualityScorer` is synchronous and completes in milliseconds.

## State machine

Seven states. The interesting paths:

- The happy path is `RECEIVED → SANITIZED → ENGAGING → MEETING_BOOKED → [terminal after quality score]`.
- Two alternate terminal paths: `DISMISSED` (low-intent lead) and `HANDED_OFF` (enterprise lead escalated to a human AE). Both transition after the quality score lands.
- Two failure transitions land in `FAILED`: a sanitizer error during `RECEIVED`, and an agent or guardrail-exhaustion error during `ENGAGING`. Prior entity state is preserved on a `FAILED` lead for ops review.
- There is no `CONVERTED` or `CLOSED_WON` state. The blueprint covers the handoff, not the full sales cycle.

## Entity model

`LeadEntity` is the source of truth. It emits nine event types. `LeadView` projects every event into a row for the UI. `LeadSanitizer` subscribes to entity events to strip PII and produce the sanitized contact. `LeadWorkflow` both reads (`getLead`) and writes (`startEngagement`, `recordTurn`, `recordBooking`, `dismiss`, `handoffToAE`, `recordQuality`, `fail`) on the entity. The relationship between `SdrAgent` and `AgentTurn` is "returns" — the agent's task result is the turn record, which may carry a tool call.

## Defence-in-depth governance flow

For any turn that lands in the entity log, the message passed through:

1. **PII sanitizer** — the model works on a redacted contact record; the audit log retains the raw form.
2. **SdrAgent** — one model call, one structured output per turn.
3. **before-agent-response guardrail (ReplyGuardrail)** — pricing commitments, competitor names, and empty replies are caught before the message is recorded.
4. **before-tool-call guardrail (BookingGuardrail)** — out-of-bounds calendar slots and invalid CRM field values are caught before write actions are dispatched.
5. **Quality scorer** — after the conversation closes, a deterministic score surfaces conversations where the agent pushed to close too early or missed discovery steps.

Each check is independent. Removing one opens an explicit gap the others do not silently cover.
