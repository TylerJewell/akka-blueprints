# User journeys — safety-plugins

Numbered acceptance journeys. Each defines preconditions, steps, and expected outcomes that the generated system must satisfy.

---

## J1 — Happy-path screening (benign payload, GENERAL profile)

**Preconditions:**
- Service is running (`/akka:build` completed successfully).
- The seeded GENERAL safety profile (8 rules) and the benign user-message payload are available from the App UI.

**Steps:**
1. Open the App UI tab.
2. Select profile `GENERAL` from the dropdown.
3. Click "Load seeded example" to populate the payload textarea with the benign user message seed.
4. Enter `test-operator-1` in Submitted by.
5. Click **Submit for screening**.
6. Observe the new card in the live list.

**Expected:**
- The card appears in `SUBMITTED` state within 1 s of the click.
- Within 2 s, it transitions to `SANITIZED`. The right-pane preview shows the redacted form; zero or more PII category chips appear depending on the seeded payload's content.
- Within 30 s, the card transitions to `DECISION_RECORDED`. The overall action badge shows `ALLOW`. The findings table has one row per submitted rule, all with action `ALLOW`.
- Within 2 s, the card transitions to `EVALUATED`. The eval score chip shows 3–5. The one-line rationale confirms all ALLOW findings included justification sentences.
- The raw payload is not visible anywhere in the UI.

---

## J2 — Output guardrail retry path (malformed decision on first iteration)

**Preconditions:**
- Service is running with the Mock LLM provider (option a during `/akka:specify`).
- The mock is configured to return a malformed response on the first iteration of every 3rd screening (modulo `seedFor(screeningId)`).

**Steps:**
1. Submit three screenings in sequence using the seeded benign payload and GENERAL profile.
2. Observe the 3rd card's state transitions.

**Expected:**
- The 3rd card reaches `SCREENING` state.
- The internal agent loop calls `OutputGuardrail`, which rejects the first candidate (malformed ruleId or invalid action enum).
- The loop retries. The second iteration produces a well-formed `SafetyDecision`.
- The card transitions to `DECISION_RECORDED` with a valid overall action badge.
- The UI never shows a malformed state. `request.rawPayload` in the entity JSON does not contain any guardrail error text.
- The card eventually reaches `EVALUATED`.

---

## J3 — Low eval score for evidence-thin BLOCK decision

**Preconditions:**
- Service is running.
- The Mock LLM provider is configured to return a `SafetyDecision` with `overallAction = BLOCK` where all `CategoryFinding.evidenceQuote` fields are empty strings.

**Steps:**
1. Submit a screening that triggers the evidence-thin mock response.
2. Observe the eval score chip after `EVALUATED` lands.

**Expected:**
- The card reaches `EVALUATED`.
- The eval score chip shows `1` or `2`.
- The one-line rationale reads something like "BLOCK findings have no evidence quotes; rationale quality insufficient."
- The card border is highlighted red (score ≤ 2 threshold).
- No other screening cards are affected.

---

## J4 — PII never reaches the model

**Preconditions:**
- Service is running.
- The seeded prompt-injection payload contains `jane.doe@example.com` and `SSN 987-65-4321`.

**Steps:**
1. Submit a screening with the prompt-injection seed payload and ENTERPRISE profile.
2. After the screening reaches `EVALUATED` or `FAILED`, fetch `/api/screenings/{id}` via the browser's developer tools or curl.

**Expected:**
- `request.rawPayload` in the JSON response contains `jane.doe@example.com` and `SSN 987-65-4321` verbatim (audit log preserved).
- `sanitized.redactedPayload` contains `[REDACTED-EMAIL]` and `[REDACTED-SSN]` in place of those strings.
- `sanitized.piiCategoriesFound` includes `"email"` and `"ssn"`.
- The LLM call log (if inspectable via the Akka local console) does not contain the original PII strings.
- The payload attachment passed to `SafetyAgent` is the redacted form.

---

## J5 — Hard-block on prompt-injection input (before-llm-call guardrail)

**Preconditions:**
- Service is running.
- The seeded prompt-injection payload contains the string `IGNORE PREVIOUS INSTRUCTIONS` (one of the patterns in `injection-patterns.txt`).

**Steps:**
1. Submit a screening using the prompt-injection seed payload with any profile.
2. Observe the card state transitions.

**Expected:**
- The card reaches `SANITIZED` (PII sanitizer runs first).
- The workflow starts and the `InputGuardrail` fires during `screenStep` before any LLM call.
- The card transitions directly to `FAILED`.
- The right-pane detail shows the failure reason, e.g., `inputGuardrail.hardBlock: injection-pattern-1`.
- No `SCREENING` state is visible on the card — the transition from `SANITIZED` goes straight to `FAILED`.
- No LLM API call is recorded for this screening.

---

## J6 — Concurrent screenings maintain independent lifecycle

**Preconditions:**
- Service is running.

**Steps:**
1. Submit three screenings in rapid succession (within 2 s of each other) using different profiles: GENERAL, CHILDREN, ENTERPRISE.
2. Observe all three cards in the live list simultaneously.

**Expected:**
- All three cards appear in the live list.
- Each card progresses through its own lifecycle independently; state transitions on one card do not affect the others.
- Each card's right-pane detail shows the correct profile's rules and findings — no cross-contamination between screenings.
- All three cards reach `EVALUATED` (or `FAILED` if applicable).
- The SSE stream delivers events for all three screenings; the UI reconciles by `screeningId`.
