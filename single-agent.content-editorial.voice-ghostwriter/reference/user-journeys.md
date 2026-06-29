# User journeys — ghostwriter

Six acceptance journeys. Each defines the preconditions, steps, and expected outcome the system must satisfy for the blueprint to be considered working.

---

## J1 — Happy-path blog-post draft

**Preconditions:** Service is running. The `alex-chen` seeded corpus is loaded (3 blog-post samples).

**Steps:**
1. Open the App UI tab.
2. Select voice owner `alex-chen`, format `blog-post`, enter topic `Why async observability should be a first-class product feature`, target word count 400.
3. Click **Generate draft**.
4. Observe the live-list card transition: `SUBMITTED` → `SANITIZED` → `DRAFTING` → `DRAFT_READY`.

**Expected:**
- The transition completes within 30 s.
- The `DraftResult.draftBody` is non-empty and between 340–460 words.
- `fidelityScore` is in 0–100.
- The style markers list contains at least 2 entries, each with a non-empty `evidence` quote.
- `guardrailRejections` is 0.
- The sanitized corpus preview shows the redacted samples; no raw PII appears in the UI.

---

## J2 — Guardrail rejects first iteration and retries

**Preconditions:** Service is running with the mock LLM provider. The seed-selection algorithm is configured to return a failing response (verbatim-reproduction violation) on the first iteration of every 3rd draft.

**Steps:**
1. Submit three drafts in sequence (any voice owner, any format) until the third draft triggers the mock failing path.
2. Observe the live-list card for the third draft.

**Expected:**
- The draft still reaches `DRAFT_READY` (the second iteration succeeds).
- `DraftResult.guardrailRejections` equals 1.
- The UI displays the guardrail-rejections count chip in yellow.
- The `draftBody` on the final result does NOT contain the verbatim run that the first iteration contained.
- The entity log contains exactly one `DraftReady` event (no partial-draft events for failed iterations).

---

## J3 — PII never reaches the model

**Preconditions:** Service is running. A writing sample containing `sarah.chen@corp.example` and `EIN 94-3XXXXXX` is ready.

**Steps:**
1. Submit a new draft with a single custom writing sample containing the PII strings above.
2. Wait for `DRAFT_READY`.
3. Fetch `GET /api/drafts/{id}` and inspect the JSON.

**Expected:**
- `brief.samples[0].rawBody` in the JSON response retains the original strings `sarah.chen@corp.example` and `EIN 94-3XXXXXX` (audit trail intact).
- `sanitizedCorpus.samples[0].redactedBody` contains `[REDACTED-EMAIL]` and `[REDACTED-EIN]` in place of the originals.
- The LLM call log (if accessible in dev mode) shows only the redacted form — neither original string appears in any model input.
- `sanitizedCorpus.piiCategoriesFound` includes `"email"` and `"ein"`.

---

## J4 — All iterations fail; entity lands in FAILED

**Preconditions:** Service is running with the mock LLM provider configured to always return a failing response (e.g., `draftBody` containing `[REDACTED-EMAIL]`) for a specific `draftId` seed.

**Steps:**
1. Submit a draft that will hit the always-fail mock path.
2. Observe the live-list card status.

**Expected:**
- The card transitions through `SUBMITTED → SANITIZED → DRAFTING → FAILED`.
- The entity's `status` is `FAILED`.
- The UI card shows the final guardrail rejection reason (e.g., "Draft body contains redaction marker: [REDACTED-EMAIL]").
- No `DraftReady` event is present in the entity log.

---

## J5 — Multiple voice owners, multiple formats

**Preconditions:** Service is running. All three seeded voice owners (`alex-chen`, `morgan-riley`, `jordan-park`) have loaded corpora.

**Steps:**
1. Submit one `release-note` brief for `morgan-riley`.
2. Submit one `social` brief for `jordan-park`.
3. Wait for both to reach `DRAFT_READY`.

**Expected:**
- Both cards appear in the live list, newest-first.
- The `morgan-riley` draft body reads like a structured changelog entry; the `jordan-park` draft is under 250 words.
- The SSE stream emits both drafts' state transitions in chronological order; no events from one draft appear mixed under the other's `draftId`.

---

## J6 — Late-joining UI client sees full state

**Preconditions:** Service is running. At least two drafts are in `DRAFT_READY` state.

**Steps:**
1. Open a new browser tab to the App UI (or refresh the existing one).
2. The UI client opens a new SSE connection to `GET /api/drafts/sse`.

**Expected:**
- The live list immediately shows all existing drafts with their current status (no blank list until the next event).
- The SSE stream's first batch of events carries the full current row for each existing draft.
- The user can click any existing draft card and see its full detail — brief, sanitized corpus, draft body, style markers — without a separate fetch.
