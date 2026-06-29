# User journeys — slack-lead-qualifier

## J1 — High-score lead enriched and posted

**Preconditions:** Service running on port 9931; valid model-provider API key set; `SlackEventPoller` enabled.

**Steps:**
1. Open `http://localhost:9931/` → App UI tab.
2. Wait up to 20 s for the first simulated member event.

**Expected:**
- Lead appears with status RECEIVED, then ENRICHED within the enrichment step timeout (≤ 30 s).
- Within 1 s of ENRICHED the status transitions to SANITIZED; PII-categories-stripped chips appear on the card.
- Within the scoring step timeout (≤ 20 s) the status transitions to SCORED with tier HOT or WARM and a numeric score badge.
- For a HOT/WARM lead: status transitions to POSTING, then POSTED. The right-detail pane shows the Slack post draft (channel, headline, body) and guardrail result APPROVED.
- The simulated Slack post fires but no real HTTP call leaves the process in dev mode.

## J2 — Low-score lead suppressed without posting

**Preconditions:** Service running; canned event with COLD/DISQUALIFIED enrichment in the simulator queue.

**Steps:**
1. Wait for a simulator event that enriches to a student or freelancer profile.

**Expected:**
- Lead reaches SCORED with tier COLD or DISQUALIFIED and score < 60.
- Status transitions directly to SUPPRESSED. The detail pane shows the suppression reason `"score-below-threshold:<score>"`.
- No PostDrafted event is emitted; no Slack call is attempted.

## J3 — PII stripped before scoring LLM call (audit check)

**Preconditions:** Service running with debug logging enabled (`LOG_LEVEL=DEBUG`); the PII-rich canned event is in the simulator (rawSearchSummary contains an email address).

**Steps:**
1. Wait for the PII-rich event to be processed.
2. Inspect the service log for the LeadScoringAgent LLM call payload.

**Expected:**
- The log shows the LLM input contains `SanitizedEnrichment` with the email-address field absent and `piiCategoriesStripped` including `"email"`.
- The raw `EnrichmentResult.rawSearchSummary` (which contained the email address) is NOT present in the scoring LLM call.
- The entity's `enrichment.rawSearchSummary` (audit log) still has the raw value; the entity's `sanitized` has no identifiers.

## J4 — Guardrail blocks a post with residual PII

**Preconditions:** Service running; a test hook or integration test that injects a `SlackPostDraft` body containing an email address into `conditionalPostStep`.

**Steps:**
1. Trigger a lead workflow with a draft body that contains `bad@example.com`.

**Expected:**
- The `SlackPostGuardrail` before-tool-call hook fires.
- The check for RFC-5322 email patterns detects `bad@example.com`.
- The hook returns `GuardrailViolation{reason: "residual-pii", checkName: "email-pattern"}`.
- Status transitions to FAILED. The detail pane shows the violation reason.
- No Slack post is made; `LeadPosted` event is NOT emitted.

## J5 — Sentinel enrichment causes graceful failure

**Preconditions:** `MockModelProvider` configured to return `EnrichmentResult.unknown()` for a specific leadId.

**Steps:**
1. Submit or wait for the event that triggers the sentinel enrichment.

**Expected:**
- `PiiSanitizer` Consumer detects `company == "unknown"` and calls `LeadEntity.markFailed`.
- Status transitions to FAILED with reason `"sentinel-enrichment"`.
- `LeadWorkflow` exits cleanly; no scoring call is attempted.
- The next event in the queue is processed normally — one failed lead does not stall the pipeline.
