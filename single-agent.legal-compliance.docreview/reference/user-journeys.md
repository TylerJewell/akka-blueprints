# User journeys — docreview

## J1 — Submit a DPA and get a verdict

**Preconditions:** Service running on declared port (`http://localhost:9144/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9144/` → App UI tab.
2. From the **Instruction set** dropdown, pick `DPA (5 clauses)`.
3. Click **Load seeded example** to fill the document title and the document textarea.
4. Click **Submit for review**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted document; the PII category chips show `email`, `address` (at minimum — depending on the seed content).
- Within 30 s the card reaches `VERDICT_RECORDED`. The right pane shows: a decision badge (PASS / NEEDS_REVISION / FAIL), the summary paragraph, and a finding row for *each of the 5 submitted instructions*. Every finding has a non-empty `documentSection`, a non-empty `quote`, and a non-empty `recommendation`.
- Within 1 s of `VERDICT_RECORDED`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a malformed verdict

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `review-document.json` includes deliberately malformed entries (a finding whose `instructionId` is not in the submitted list; a severity outside the enum).

**Steps:**
1. Submit any seeded document three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/reviews/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed verdict.
- The `before-agent-response` guardrail rejects it. The malformed verdict NEVER lands in `ReviewEntity` — there is no `VerdictRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed verdict. The card transitions to `VERDICT_RECORDED` with a verdict that satisfies all four guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J3 — Evidence-thin verdict flags eval score 1

**Preconditions:** Mock LLM mode. A reviewer submits a custom instruction set whose mock entry returns findings with empty `quote` fields.

**Steps:**
1. In the App UI's Instruction set dropdown, pick `custom`. Paste a 5-instruction list whose `instructionId` values match a specific mock-response entry crafted to have empty quotes (see `src/main/resources/mock-responses/review-document.json` "evidence-thin" entry).
2. Submit any seed document.

**Expected:**
- The verdict lands well-formed (the guardrail only checks structural validity, not evidence quality).
- The eval score chip shows **1** and the rationale reads "Findings cite document sections but quote strings are empty; verdict is not anchored in the document."
- The card's border highlights red. The reviewer knows to inspect this verdict before acting.

## J4 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a document containing the literal strings `jane.doe@example.com`, `SSN 123-45-6789`, and `4111-1111-1111-1111` (a test credit-card number).
2. Wait for `VERDICT_RECORDED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/reviews/{id}` and read `request.rawDocument`.

**Expected:**
- The logged LLM call body contains the redacted forms only: `[REDACTED-EMAIL]`, `[REDACTED-SSN]`, `[REDACTED-PCN]`. The raw strings do not appear.
- `request.rawDocument` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.piiCategoriesFound` lists `email`, `ssn`, `payment-card-number` (at minimum).

## J5 — Multi-instruction completeness

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit any seeded document with the `MSA (7 clauses)` instruction set.
2. Wait for `EVALUATED`.

**Expected:**
- The verdict's `findings` array has exactly 7 entries, one per submitted instruction. No silent omissions.
- Each `findings[i].instructionId` matches one of the 7 submitted `instructionId` values, one-to-one.
- If the agent's first iteration omitted an instruction, the guardrail would have rejected it — the absence of any rejection in the log confirms the first iteration was complete.
