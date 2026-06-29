# User journeys — meeting-preparer

## J1 — Submit a meeting request and get a full brief

**Preconditions:** Service running on declared port (`http://localhost:9818/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9818/` → App UI tab.
2. From the **Load seeded counterparty** dropdown, pick `Meridian Capital Advisors`.
3. Verify the form fills: counterparty name, a meeting date/time 5 days from today, agenda topics, and requested-by.
4. Click **Prepare brief**.

**Expected:**
- The new card appears in the live list with status `REQUESTED` within 1 s.
- The card transitions to `DATA_SANITIZED` within 1 s. The right-pane detail shows the redacted CRM snapshot; the PII category chips show `person-name`, `email`, `phone` (at minimum).
- Within 30 s the card reaches `BRIEF_READY`. The right pane shows: an executive summary paragraph, at least 3 talking points each with a non-empty `evidenceSource`, at least 1 risk flag, a financial highlights line, and at least 2 recent news items.
- Within 1 s of `BRIEF_READY`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a meeting request whose `crmSnapshot.contactEmail` is `test.contact@counterparty.example.com` and `crmSnapshot.contactPhone` is `+1-415-555-0199`.
2. Wait for `BRIEF_READY`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment` for `sanitized-crm.txt`).
4. Fetch `GET /api/briefs/{id}` and read `request.crmSnapshot.contactEmail`.

**Expected:**
- The logged LLM call body for `sanitized-crm.txt` contains `[REDACTED-EMAIL]` and `[REDACTED-PHONE]`. The raw strings do not appear.
- `request.crmSnapshot.contactEmail` in the JSON response still contains `test.contact@counterparty.example.com` — the audit log preserves the raw value.
- `sanitized.piiCategoriesFound` lists `email` and `phone` (at minimum).

## J3 — Evidence-thin brief flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `prepare-meeting-brief.json` includes a deliberately incomplete entry with empty `evidenceSource` strings on all talking points.

**Steps:**
1. Submit the seeded counterparty whose mock entry maps to the "evidence-thin" response (per `MockModelProvider.seedFor(briefId)` — this is the entry selected on every fourth request modulo seed).
2. Wait for `EVALUATED`.

**Expected:**
- The brief lands well-formed (the entity accepts it because structural validity is not gated by a guardrail in this blueprint; the evaluator scores completeness, not structure).
- The eval score chip shows **1** and the rationale reads "Talking points carry no evidence sources; brief is not grounded in the attachments."
- The card's border highlights red.

## J4 — Three concurrent requests update via SSE independently

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Open two browser tabs both pointed at `http://localhost:9818/` → App UI tab.
2. In tab 1, submit a meeting request for `Meridian Capital Advisors`.
3. Immediately submit a second request for `Northridge Insurance Group`.
4. Immediately submit a third request for `Ashford Manufacturing Holdings`.

**Expected:**
- All three cards appear in the live list within 2 s across both browser tabs.
- Each card transitions through its lifecycle independently — the three workflows run concurrently.
- No card's status update in tab 1 is blocked by another card's agent call completing in a different order.
- When all three reach `EVALUATED`, the live list shows all three with green status pills and distinct eval scores.

## J5 — Agenda-topic coverage in talking points

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a meeting request with three agenda topics: `"regulatory update"`, `"capital structure"`, `"Q3 outlook"`.
2. Wait for `BRIEF_READY`.

**Expected:**
- The brief's `talkingPoints` list contains at least one entry for each of the three agenda topics in its `topic` field.
- If an agenda topic has no grounding in the attachments, the talking point for that topic explicitly notes the absence of background material rather than fabricating content.
