# Architecture

The generated system's Architecture tab renders the four mermaid diagrams from PLAN.md with the following narrative.

## Component graph
`KnowledgeEndpoint` is the entry point. A submitted inquiry is logged as `InquirySubmitted` on `IntakeQueue` (event-sourced for audit). `InquiryRequestConsumer` subscribes to the queue, creates the `InquiryEntity`, and starts a `ControlShell` workflow for that inquiry. The shell is the only component that reads-then-writes the `BlackboardEntity`; the four specialist agents (`SourceScout`, `ClaimExtractor`, `HypothesisSynthesizer`, `Critic`) never call each other and never write directly. Every transition on the blackboard projects into `BlackboardView` for the UI's SSE feed, and `ContributionScorer` subscribes to the same events and writes back a score per accepted write. `SystemControl` holds the operator halt flag, which the shell consults on every tick and which the schema guardrail consults on every commit. Two TimedActions sit alongside: `InquirySimulator` drips canned questions; `StaleInquiryMonitor` closes inquiries that have not advanced.

## Interaction sequence
The sequence diagram traces the happy path (J1): submit → control shell loop → specialist invocation → schema guardrail → commit → score → repeat → converge. Note where the schema guardrail vets every `ProposedWrite` before the entity commits; on rejection the entity records `WriteRejected` and state is unchanged, but the rejection is visible in the projection so the operator can see which specialists are misbehaving.

## State machine
`BlackboardEntity` is the heart of the pattern. `OPEN` is the initial state; every accepted write keeps the entity in `OPEN` and updates the structured payload. `Converged` is the terminal-but-not-closed state recorded once a non-disputed hypothesis covers every open sub-question and the iteration budget still holds. `Closed` is the post-mortem state used both when an inquiry converges and is summarised and when the operator closes it manually or `StaleInquiryMonitor` closes it for inactivity.

## Entity model
`BlackboardEntity` is the source of truth for an inquiry's working knowledge; every accepted contribution writes one of ten event types and `BlackboardView` is the only read-side projection — `ControlShell` reads the entity directly because it has to act on the freshest state, but the UI reads the view. `InquiryEntity` tracks lifecycle; `IntakeQueue` is the audit log of submissions; `ContributionScorer` writes scores back to the same blackboard so the control shell can read score history when selecting the next specialist.

## Concurrency & timeouts
- Single-writer on the blackboard is the coordination primitive. There is no shared in-memory state, no lock, no external queue — every specialist's contribution flows through the entity.
- Per-step timeout: 90 s on the agent-calling step (`ControlShell.invokeSpecialistStep`).
- Idle inquiries are paused workflows with a 5 s resume timer, not busy loops.
- Specialist selection consults the latest 3 contribution scores per specialist; a benched specialist returns to play when a new sub-question opens or the critic flags an item.
- `StaleInquiryMonitor` closes inquiries whose `lastWriteAt` is older than 5 min so a stalled control shell does not strand the inquiry forever.
- The schema guardrail is the only write gate; it consults `SystemControl` so a halt blocks every commit immediately.
