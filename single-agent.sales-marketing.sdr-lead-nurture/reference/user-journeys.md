# User journeys — sdr-lead-nurture

## J1 — Inbound lead reaches a booked meeting

**Preconditions:** Service running on declared port (`http://localhost:9409/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9409/` → App UI tab.
2. Click **Load seeded example** to fill the ingest form with the mid-funnel website-chat lead.
3. Click **Start engagement**.
4. The card appears in the live list with status `RECEIVING` within 1 s, then transitions to `SANITIZED`. The right pane shows the redacted contact record with PII category chips (`email`, `phone`, `person-name` at minimum).
5. Within ~5 s the card transitions to `ENGAGING`. The conversation thread shows the agent's first reply — a discovery question.
6. In the reply composer, type a follow-up message expressing intent ("We're ready to evaluate — can we set up a call?") and click **Send**.
7. Within ~10 s the agent responds with a meeting proposal and the card transitions to `MEETING_BOOKED`. The booking card shows the slot, duration, and a synthetic Zoom link.
8. The quality score chip appears within 1 s of `MEETING_BOOKED`.

**Expected:**
- Every AGENT turn in the conversation passed the brand-safety guardrail — no pricing deny-list phrase or competitor brand appears in any reply.
- The meeting slot is a weekday between 08:00 and 17:00.
- The quality score is ≥ 3 for a 2-turn conversation that included at least one discovery question.

## J2 — Brand-safety guardrail blocks a non-compliant draft

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `sdr-engage.json` includes a deliberately non-compliant entry containing a pricing deny-list phrase.

**Steps:**
1. Ingest a lead (J1 steps 2–3).
2. Watch the SSE stream (`/api/leads/sse`) in the browser dev tools network panel.
3. Observe the entity event log — look for `TurnRecorded` events.

**Expected:**
- The mock's first candidate reply for this lead contains a prohibited pricing phrase.
- The `before-agent-response` guardrail rejects it. No `TurnRecorded` event with that message ever lands on `LeadEntity`.
- The agent retries on iteration 2 and produces a compliant reply. The `TurnRecorded` event for iteration 2 appears in the SSE stream; the conversation thread shows the compliant reply.
- The service log shows one `guardrail.reject[brand-violation]` line naming the prohibited phrase.

## J3 — Before-tool-call guardrail blocks an out-of-hours booking

**Preconditions:** Mock LLM mode. The mock includes a `BOOK_MEETING` tool call where the slot is 07:00 on a Saturday.

**Steps:**
1. Ingest the mid-funnel lead.
2. Send a reply that signals booking intent ("Let's book a call for this weekend.").
3. The mock's agent turn responds with a `BOOK_MEETING` call for `2026-07-05T07:00:00` (a Saturday at 07:00).
4. Watch the SSE stream and service log.

**Expected:**
- `BookingGuardrail` blocks the call with a `tool-blocked` error citing "slot not within weekday 08:00–17:00".
- No `MeetingBooked` event is emitted for the out-of-hours slot.
- The agent's next iteration proposes a weekday slot within allowed hours; `BookingGuardrail` accepts it.
- `MeetingBooked` lands on the entity with the corrected slot.

## J4 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Fill the ingest form manually with `firstName = "Taylor"`, `email = "taylor.chen@example.com"`, `phone = "+44-7700-900123"`.
2. Start engagement and wait for `ENGAGING`.
3. Inspect the service log for the LLM call body (`debug:agent.task.instructions`).
4. Fetch `GET /api/leads/{id}` and read `contact.email` and `contact.phone`.

**Expected:**
- The logged LLM call body contains `[REDACTED-EMAIL]` and `[REDACTED-PHONE]` in place of the raw values. The raw strings do not appear.
- `contact.email` and `contact.phone` in the JSON still contain the raw values — the entity audit log preserves them.
- `sanitized.piiCategoriesFound` lists `email`, `phone`, `person-name` (at minimum).

## J5 — Low-intent lead is dismissed

**Preconditions:** Mock LLM mode. The mock includes a `DISMISS_LEAD` tool-call entry for the low-intent seeded lead.

**Steps:**
1. Load the low-intent seeded example (channel = `EVENT`, initial message signals a student project).
2. Start engagement.
3. Observe the conversation through to close.

**Expected:**
- The agent's close action is `DISMISS_LEAD` with a polite reason string.
- The entity transitions to `DISMISSED`.
- The quality score chip appears. The outbound message in the final turn does not contain any unapproved pricing or competitor references.
- The card in the live list shows the `DISMISSED` status chip.

## J6 — Enterprise lead is handed off to an AE

**Preconditions:** Mock LLM mode. The mock includes a `HANDOFF_TO_AE` tool-call entry for the enterprise seeded lead.

**Steps:**
1. Load the enterprise seeded example (company has > 500 employees, initial message mentions "enterprise procurement").
2. Start engagement.
3. Observe the conversation through to close.

**Expected:**
- The agent's close action is `HANDOFF_TO_AE` with `priority = HIGH`.
- The entity transitions to `HANDED_OFF`.
- No `BOOK_MEETING` is attempted for an enterprise lead (the agent escalates rather than self-booking).
- The quality score chip appears; the rationale notes the proportional escalation.
