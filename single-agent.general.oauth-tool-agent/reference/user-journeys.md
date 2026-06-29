# User journeys — ae-oauth

## J1 — Full-access token: all tool calls allowed

**Preconditions:** Service running on declared port (`http://localhost:9138/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9138/` → App UI tab.
2. From the **Token** dropdown, pick `full-access`.
3. Click **Load example** — the request textarea fills with: "List my calendar events, list my files, and add a new calendar event for tomorrow at 3 PM."
4. Click **Run**.

**Expected:**
- The new session card appears with status `PENDING` within 1 s.
- The card transitions to `TOKEN_RESOLVED` within 1 s. The right-pane detail shows the resolved scope chips: `calendar:read`, `calendar:write`, `files:read`, `files:write`, `contacts:read`.
- The card transitions to `RUNNING` as the agent starts.
- Within 30 s the card reaches `COMPLETED` with `outcome: SUCCESS`. The tool calls table shows three rows: `listCalendarEvents` (ALLOWED), `listFiles` (ALLOWED), `createCalendarEvent` (ALLOWED). Each ALLOWED row shows non-empty output from `SimulatedToolExecutor`.
- The agent summary mentions all three actions completed.

## J2 — Read-only token: write call denied

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Token** dropdown, pick `read-only`.
2. Type (or load): "Create a new calendar event for Monday at 9 AM."
3. Click **Run**.

**Expected:**
- Session card reaches `TOKEN_RESOLVED` with scopes `calendar:read`, `files:read`, `contacts:read` — no write scopes visible.
- The agent proposes `createCalendarEvent`. The `before-tool-call` guardrail checks `calendar:write` against the token's scope set, finds it absent, and returns a denial.
- The session reaches `COMPLETED` with `outcome: DENIED`. The tool calls table shows one row: `createCalendarEvent` (DENIED) with denial reason "token lacks scope calendar:write".
- The agent summary states the event could not be created due to insufficient scope, rather than fabricating a success.

## J3 — Calendar-only token: partial scope coverage

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Token** dropdown, pick `calendar-only`.
2. Type (or load): "List my calendar events and also list my files."
3. Click **Run**.

**Expected:**
- Session card reaches `TOKEN_RESOLVED` with scopes `calendar:read`, `calendar:write` only.
- The agent proposes `listCalendarEvents` → guardrail allows (`calendar:read` present) → `SimulatedToolExecutor` returns event list.
- The agent proposes `listFiles` → guardrail denies (`files:read` absent).
- Session reaches `COMPLETED` with `outcome: PARTIAL`. Tool calls table shows: `listCalendarEvents` (ALLOWED) and `listFiles` (DENIED).
- The agent summary correctly distinguishes which action succeeded and which was blocked.

## J4 — Expired token: session fails before agent is invoked

**Preconditions:** Service running. The `expired-token` token id is in `TokenRegistry` with `expiresAt: 2020-01-01`.

**Steps:**
1. From the **Token** dropdown, pick `custom` and type the token id `expired-token`.
2. Type any request text.
3. Click **Run**.

**Expected:**
- The session card appears with status `PENDING` and transitions directly to `FAILED` within 1 s — no `TOKEN_RESOLVED` transition occurs.
- The right-pane detail shows the failure reason: "token expired".
- No agent call is made — the service log contains no `agent.task` line for this session.
- The session card border is red, consistent with the `FAILED` status colour.

## J5 — Multi-step request: sequential tool calls with mixed outcome

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Token** dropdown, pick `read-only`.
2. Type: "List my contacts, list my calendar events, and upload a file named report.pdf."
3. Click **Run**.

**Expected:**
- Session resolves scopes: `calendar:read`, `files:read`, `contacts:read`.
- Agent proposes three calls in sequence. Guardrail allows `listContacts` (`contacts:read` present), allows `listCalendarEvents` (`calendar:read` present), and denies `uploadFile` (`files:write` absent).
- Session reaches `COMPLETED` with `outcome: PARTIAL`. Tool calls table shows three rows: two ALLOWED, one DENIED.
- Order of rows in the tool calls table matches the order the agent proposed them, not alphabetical order.
