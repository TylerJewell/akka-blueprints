# User journeys — inbound-lead-qualification

## J1 — Submit a lead and receive a HOT qualification with Slack notification

**Preconditions:** Service running on declared port (`http://localhost:9310/`); a valid model-provider API key set or mock LLM selected at scaffold time; `SLACK_BOT_TOKEN` set or `MockSlackClient` selected. The seeded lead `Quantum Systems / CTO / 1 001+ employees` has a matching `src/main/resources/sample-data/firmographics/quantumsystems.io.json` file.

**Steps:**
1. Open `http://localhost:9310/` → App UI tab.
2. From the **Pick a seeded lead** dropdown, select `Quantum Systems / CTO / 1 001+ employees`.
3. Click **Submit lead**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s, then `ENRICHING` within 1 s more.
- Within ~20 s the card reaches `ENRICHED`. The right pane shows the Enrichment panel with a non-empty `industry`, `estimatedArrBand`, `employeeBand`, and at least two `techStack` signals.
- Within ~20 s more the card reaches `QUALIFIED`. The right pane shows a `QualificationScore` with `tier = HOT`, `score ≥ 70`, a non-blank `assignedRepName`, and a non-blank `assignedRepSlackId`.
- Within ~20 s more the card reaches `NOTIFIED`. The right pane shows the Slack notification panel with a non-empty `channel`, `text`, and at least two `blocks`.
- Within ~1 s the card reaches `EVALUATED` with `confidence = HIGH`.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — SlackGuardrail blocks a premature Slack write

**Preconditions:** Service running with mock LLM selected (`model-provider = mock`). The mock's `notify-sales.json` includes one entry whose `tool_calls` array starts with `postLeadToSlack` before `buildSlackMessage` — this is the deliberately out-of-sequence entry, selected on the first iteration of every 3rd lead (modulo seed).

**Steps:**
1. Submit any seeded lead three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/leads/sse`).

**Expected:**
- On the third submission's `notifyStep`, the agent's first iteration calls `postLeadToSlack` when the entity's `score` is not yet present in its NOTIFYING context. `SlackGuardrail` rejects the call; a `GuardrailRejected{phase: "NOTIFY", tool: "postLeadToSlack", reason: "slack-write-blocked: ..."}` event lands on the entity.
- The rejected call NEVER reaches `NotifyTools.postLeadToSlack` — there is no Slack API call or mock-client log line from the tool body.
- The agent's second iteration calls `buildSlackMessage` first, then `postLeadToSlack`. The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — LOW confidence eval flags a score-tier mismatch

**Preconditions:** Mock LLM mode. The mock's `qualify-lead.json` includes one entry with `score = 80` and `tier = COLD` — a deliberately misaligned pairing, selected on the modulo-seed run.

**Steps:**
1. Submit any seeded lead six times. Watch the live list.
2. When a card's border highlights amber, select it.

**Expected:**
- The flagged lead is well-formed and reaches `NOTIFIED` (the guardrail only checks Slack-write ordering, not score-tier alignment).
- The eval confidence chip shows **LOW** and the rationale reads: *"Score-tier alignment check failed: score 80 is consistent with tier HOT, not COLD."*
- The card's border highlights amber. The sales operations team knows to review this lead before the rep acts on it.
- The other five leads in the run have `confidence = HIGH` or `MED`.

## J4 — PII does not appear in enrichment-tool context

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit the seeded lead `Bright Futures / Head of Product / 22 employees`.
2. Wait for `ENRICHED`.
3. Inspect the service log for ENRICH-task lines. Filter to entries tagged with the leadId.

**Expected:**
- The ENRICH task's instruction string in the log contains pseudonymous tokens (`[NAME-1]`, `[EMAIL-1]`) — no raw first name, last name, or email address.
- `lookupFirmographics` and `fetchTechStack` log entries show only the company domain (`brightfutures.co`) — no personal data.
- The original `firstName`, `lastName`, and `email` ARE present when `GET /api/leads/{id}` is called — the entity retains the full `LeadFormData`.
- No line in the ENRICH phase log contains the string `@` (no email present in agent context).

## J5 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded lead.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines. Filter to entries tagged with the leadId.

**Expected:**
- The ENRICH task's log shows only `lookupFirmographics` and `fetchTechStack` calls.
- The QUALIFY task's log shows only `scoreByRubric` and `assignRep` calls.
- The NOTIFY task's log shows only `buildSlackMessage` and `postLeadToSlack` calls.
- No cross-phase calls appear.
- The order of tasks in the log is ENRICH → QUALIFY → NOTIFY. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J6 — Unknown company domain produces an honest empty profile

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/firmographics/<domain>.json` file exists for the submitted company.

**Steps:**
1. In the App UI, type a custom company name (e.g. `Artisanal Pickle Co.`) and a fake domain email.
2. Click **Submit lead**.

**Expected:**
- `EnrichTools.lookupFirmographics` returns an empty profile (`industry = "(unknown)"`, `estimatedArrBand = "(unknown)"`, `employeeBand = "(unknown)"`, `techStack = []`).
- The agent's ENRICH task returns this empty `FirmographicProfile` without inventing data.
- The workflow advances to `qualifyStep`. With no employee band or ARR estimate, `scoreByRubric` assigns `tier = COLD` and `score < 40`.
- `assignRep` still runs and returns a rep name (round-robin assignment does not require enrichment data).
- `notifyStep` runs normally; the Slack message notes `"(unknown industry)"` in the section body.
- The lead reaches `EVALUATED` with `confidence = MED` (rep-assignment passes; tier-by-size is indeterminate for unknown band; score-tier alignment passes for COLD with score < 40).
- Nothing crashes; the pipeline completes with an honest partial result.
