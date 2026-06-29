# User journeys — java-bug-assistant

## J1 — Submit a bug report and get a recommendation

**Preconditions:** Service running on declared port (`http://localhost:9739/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9739/` → App UI tab.
2. From the **Bug category** dropdown, pick `runtime-error`.
3. Click **Load seeded example** to fill the title and bug report textarea with the null-pointer exception seed.
4. Click **Submit bug**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `NORMALIZED` within 1 s. The right-pane detail shows the cleaned report; the redaction category chips show `credential`, `internal-hostname` (at minimum — depending on the seed content).
- The card transitions to `SEARCHING` then `SEARCH_COMPLETE` within 2 s. The related tickets panel shows at least 1 matched ticket.
- Within 30 s the card reaches `RECOMMENDATION_RECORDED`. The right pane shows: an action badge (RESOLVED / NEEDS_INVESTIGATION / ESCALATE), the rationale paragraph, and a ranked candidate-fix list with at least 1 entry. Every fix has a non-empty `description`, a `confidenceScore` between 0 and 100, and a non-empty `sourceRef`.
- Within 1 s of `RECOMMENDATION_RECORDED`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a malformed ticket write

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `resolve-bug.json` includes deliberately malformed entries (one with an empty `candidateFixes` list; one with a `confidenceScore` outside `0..100`).

**Steps:**
1. Submit any seeded bug report three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the browser dev tools network panel (`/api/bugs/sse`).

**Expected:**
- The third submission's first agent iteration attempts a ticket write with a malformed recommendation.
- The `before-tool-call` guardrail blocks the write. The malformed payload NEVER lands in `BugEntity` — there is no `RecommendationRecorded` event with the malformed content.
- The agent loop revises its plan on iteration 2 (and 3 if needed) and produces a well-formed recommendation. The card transitions to `RECOMMENDATION_RECORDED` with a recommendation that satisfies all four guardrail checks.
- The service log shows one `guardrail.reject` line per blocked iteration with the structured-error code naming which check failed.

## J3 — Low-confidence recommendation flags eval score 1

**Preconditions:** Mock LLM mode. A submission matches a mock entry where all `candidateFixes` have `confidenceScore == 0`.

**Steps:**
1. Submit the `resource-exhaustion` seeded bug (its mock entry is the all-zero-confidence entry crafted for this journey).
2. Wait for `EVALUATED`.

**Expected:**
- The recommendation lands well-formed (the guardrail only blocks writes where confidence values are out of range, not where they are uniformly low).
- The eval score chip shows **1** and the rationale reads "All candidate fixes have zero confidence; recommendation provides no actionable guidance."
- The card's border highlights red. The engineer knows to treat this recommendation with caution.

## J4 — Sensitive data never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the normalizer runs either way).

**Steps:**
1. Submit a bug report containing the literal strings `DB_PASSWORD=s3cr3t`, `host: db-prod-01.internal`, and `eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyIn0.abc` (a JWT-like token).
2. Wait for `RECOMMENDATION_RECORDED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/bugs/{id}` and read `report.rawReport`.

**Expected:**
- The logged LLM call body contains the redacted forms only: `[REDACTED-CREDENTIAL]`, `[REDACTED-HOST]`, `[REDACTED-TOKEN]`. The raw strings do not appear.
- `report.rawReport` in the JSON still contains the raw strings — the audit log preserves them.
- `normalized.redactionCategories` lists `credential`, `internal-hostname`, `stack-trace-token` (at minimum).

## J5 — Network-failure category routes through search correctly

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the `network-failure` seeded bug report (HTTP 502 intermittent failure seed).
2. Wait for `SEARCH_COMPLETE`.

**Expected:**
- The related tickets panel shows at least 1 ticket matching the network-failure category from the seeded ticket store.
- All matched tickets have `ticketStatus` set to one of `OPEN`, `IN_PROGRESS`, `RESOLVED`, `CLOSED`, or `WONT_FIX` — no null or unknown values.
- The workflow advances to `RESOLVING` without error; the agent receives the search results as its instruction text.

## J6 — No matching tickets still produces a valid recommendation

**Preconditions:** Service running with the mock LLM. Submit a bug report whose content matches no seeded tickets (e.g., a completely novel error message).

**Steps:**
1. Submit a bug report with a unique title and error message that does not match any keyword in `seeded-tickets.jsonl`.
2. Wait for `EVALUATED`.

**Expected:**
- `searchResults` on the bug entity is an empty list (or a list of low-relevance tickets).
- The agent still produces a `ResolutionRecommendation` — the `ESCALATE` path is taken with a single candidate fix citing `sourceRef = "kb-escalation"`.
- The recommendation passes the guardrail (non-empty fix list, non-empty rationale, valid confidence score).
- The eval score may be low (2 or 3) because the fix cites a generic KB entry, but the card is not flagged red unless score ≤ 2.
