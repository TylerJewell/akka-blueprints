# Architecture — slack-lead-qualifier

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`SlackEventPoller` is the heartbeat — a TimedAction ticking every 20 s that writes simulated `MemberJoinedReceived` events into `SlackEventQueue` (event-sourced for audit). Each event starts a `LeadWorkflow` instance. The workflow calls `LeadEnrichmentAgent` (a typed Agent performing web search), emits `LeadEnriched` on `LeadEntity`, then waits. A `PiiSanitizer` Consumer subscribes to `LeadEnriched` events, strips personal identifiers from the enrichment result, and emits `LeadSanitized`. The workflow polls the entity until it sees `SANITIZED`, then calls `LeadScoringAgent`. If the score meets the configured threshold the workflow calls the `postToSlack` tool — the `SlackPostGuardrail` before-tool-call hook inspects the draft post before the call fires.

## Interaction sequence

The sequence traces the J1 happy path: a high-scoring lead joins the channel and a post is made. Two distinct waits appear: (1) the enrichment web-search call (bounded by a 30 s step timeout), and (2) the `waitSanitizedStep` poll loop (3 s interval, no hard timeout). The guardrail check at step 14–15 is a synchronous in-process call — it adds no observable latency but provides the deterministic last gate before any external write.

## State machine

Eight states. The branching points of interest:

- After `ENRICHED`, the `PiiSanitizer` Consumer can either emit `LeadSanitized` (normal path → `SANITIZED`) or, if it detects the sentinel enrichment, emit `LeadFailed` (→ `FAILED` terminal). This prevents a sentinel-only profile from reaching the scoring LLM.
- After `SCORED`, the score comparison branches to `POSTING` (score >= threshold) or `SUPPRESSED` terminal.
- From `POSTING`, the guardrail either passes (→ `POSTED` terminal) or fails (→ `FAILED` terminal with violation reason attached).

## Entity model

`LeadEntity` is the single source of truth, emitting eight distinct event types. `SlackEventQueue` is the upstream audit log — only the workflow reads from it directly; `PiiSanitizer` subscribes to `LeadEntity` events. This means the raw `MemberJoinedEvent` (including the original email) is durably stored in `SlackEventQueue` for compliance purposes while the sanitized profile is what flows into scoring and posting.

## Defence-in-depth governance flow

For any Slack post that goes out, the lead passed through:
1. **PII sanitizer** — personal identifiers stripped from enrichment before the scoring model sees the profile.
2. **LeadScoringAgent** — score must meet the threshold; disqualified leads are suppressed without any external write.
3. **SlackPostGuardrail** — tone check + residual PII scan on the draft post body at the tool boundary.

Each check is independent. Removing any one of them does not silently degrade the system; it explicitly opens a specific risk surface that the other two do not cover.
