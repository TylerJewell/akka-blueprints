# User journeys — per-tenant-agent-fleet

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path: register to READY with a chat reply

**Preconditions:** Service running on port 9512; a model-provider key set, or the mock LLM selected during scaffolding.

**Steps:**
1. Open `http://localhost:9512/`. The App UI tab is visible.
2. In the registration form, enter name "Aiko Tanaka", email "aiko@example.com", tier "PREMIUM". Click Register.
3. A `customerId` is returned; the customer card appears with status `REGISTERED`.

**Expected:**
- The fleet coordinator records a fleet entry; the card moves to `ONBOARDING`.
- The background researcher builds a profile; the card shows the profile summary.
- The welcome email is dispatched; the card shows "Welcome sent" and moves to `READY`.
- A fleet eval score chip appears on the card.
- The chat panel is now active. Enter "What does my PREMIUM tier include?" and click Send.
- A reply appears in the chat panel within the `chatStep` timeout (60 s). The reply references the customer's tier or profile context.
- The fleet board updates live via SSE throughout, with no page reload.

## J2 — Isolation: two simultaneous registrations stay separate

**Preconditions:** As J1.

**Steps:**
1. Register two customers back-to-back: "Aiko Tanaka / PREMIUM" and "Miguel Ferreira / ENTERPRISE".

**Expected:**
- Each customer gets its own card on the fleet board with its own status progression.
- The profile summary on Aiko's card contains Aiko's details; Miguel's contains Miguel's. No cross-contamination.
- Chat messages sent to Aiko's chat panel produce replies that reference Aiko's memory, not Miguel's.

## J3 — Guardrail: welcome email blocked for disallowed recipient

**Preconditions:** As J1. Under the mock LLM, a `chat-responder.json` entry whose `sendWelcomeEmail` call targets a domain in the blocked list (e.g., `example-blocked.com`); or register a customer whose email domain is in the blocked list.

**Steps:**
1. Register a customer with email "user@example-blocked.com".

**Expected:**
- The before-tool-call guardrail refuses the `sendWelcomeEmail` call.
- No email is dispatched. The `WelcomeBlockRecorded` event is written to `AgentMemoryEntity`.
- The customer card shows "Welcome blocked: <reason>" in red.
- The customer's agent still reaches `READY` and can answer chat messages.

## J4 — PII sanitizer: raw sensitive field stripped before agent sees it

**Preconditions:** As J1.

**Steps:**
1. Send a chat message to a READY customer that contains a pattern the sanitizer recognizes (e.g., a string resembling "SSN: 123-45-6789").

**Expected:**
- `PiiSanitizer` strips the SSN-like pattern at the service boundary before the message reaches `AgentFleetWorkflow` or `ChatResponder`.
- The `ChatTurn` persisted in `AgentMemoryEntity` does not contain the raw sensitive string.
- The agent's reply does not reference or echo the raw sensitive value.
- The sanitized turn is visible in the chat panel; the replacement placeholder (e.g., "[REDACTED]") appears in place of the original pattern.

## J5 — Fleet health: stale agent surfaces in health panel

**Preconditions:** A customer's agent has been in `READY` state with no chat interaction for longer than the configured stale window. Under the mock LLM or in a time-forwarded test, the `StaleAgentMonitor` has run at least once since the threshold elapsed.

**Steps:**
1. Open the App UI. Scroll to the Fleet Health panel below the fleet board.

**Expected:**
- The health panel shows a `FleetHealthSummary` for the dormant customer, including their name, the days-since-last-interaction count, and a recommendation phrase.
- `GET /api/fleet/health` returns the same summary.
- The customer's `CustomerRegistry` status remains `READY` — the monitor does not change agent state.
- No other customer (with recent interaction) appears in the health panel.
