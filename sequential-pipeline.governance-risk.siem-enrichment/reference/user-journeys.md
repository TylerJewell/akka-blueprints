# User journeys — siem-enrichment

## J1 — Submit a SIEM alert and receive a triage ticket

**Preconditions:** Service running on declared port (`http://localhost:9384/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded alert `lateral-movement-01` has a matching `src/main/resources/sample-data/alerts/lateral-movement-01.json` file.

**Steps:**
1. Open `http://localhost:9384/` → App UI tab.
2. From the **Pick a seeded alert** dropdown, pick `lateral-movement-01`.
3. Click **Run pipeline**.

**Expected:**
- The new card appears in the live list with status `RECEIVED` within 1 s, then `FETCHING` within 1 s more.
- Within ~20 s the card reaches `FETCHED`. The right pane shows the Alert detail table with ≥ 3 raw SIEM fields; `ruleName`, `severity`, `sourceIp`, `destinationIp`, and `hostname` are all non-empty.
- Within ~20 s more the card reaches `ENRICHED`. The right pane shows ≥ 1 attack pattern with a valid `techniqueId` (e.g., `T1021.002`) and a non-empty `tactic`; the derived-severity badge matches or exceeds the raw severity.
- Within ~20 s more the card reaches `TRIAGED`, then `EVALUATED` within 1 s of that. The right pane shows a `TriageTicket` with a non-empty `assignedTeam`, a Zendesk ticket link, and a summary paragraph. The eval score chip shows 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Ticket-scope guardrail blocks a premature Zendesk write

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `enrich-alert.json` includes one entry whose `tool_calls` array starts with a TRIAGE-phase tool (`openZendeskTicket`) — the deliberately phase-violating entry.

**Steps:**
1. Submit any seeded alert three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/alerts/sse`).

**Expected:**
- On the third submission's `enrichStep`, the agent's first iteration calls `openZendeskTicket`. `TicketScopeGuardrail` rejects it; a `GuardrailRejected{phase: "ENRICH", tool: "openZendeskTicket", reason: "scope-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `TicketTools.openZendeskTicket` — there is no log line from the tool body, and no stub Zendesk ticket is created.
- The agent's second iteration falls through to a normal enrich sequence (`lookupMitreTechnique` + `mapAttackPattern`). The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — Unrecognised ATT\&CK technique flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `create-ticket.json` includes one entry whose first `AttackPattern` carries a `techniqueId` absent from the bundled ATT\&CK registry (`src/main/resources/sample-data/mitre-techniques.json`).

**Steps:**
1. Submit any seeded alert six times. (The invalid-technique entry is selected once in every six runs by the mock's `seedFor(alertId)` modulo.)
2. Watch the live list until a card's border highlights red.

**Expected:**
- The flagged alert lands with a well-formed `TriageTicket` (the guardrail only checks phase order, not technique validity).
- The eval score chip shows **1** (or 2 depending on which other rules pass) and the rationale reads: *"Technique validity failed: attackPattern 'ap-...' has techniqueId 'TX999.999' which does not resolve in the ATT&CK registry."*
- The card's border highlights red. The SOC analyst knows to inspect this ticket before triggering a response playbook.
- The other five alerts in the run scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit the seeded alert `privilege-escalation-02`.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the alertId.

**Expected:**
- The FETCH task's log entries show only `fetchAlertDetail` and `fetchHostContext` calls.
- The ENRICH task's log entries show only `lookupMitreTechnique` and `mapAttackPattern` calls.
- The TRIAGE task's log entries show only `openZendeskTicket` and `assignTicketTeam` calls.
- No cross-phase calls appear. (If the mock-LLM violation path fires, a single `guardrail.reject` line precedes the agent's retry; the rejected call is logged but never executed.)
- The order of tasks in the log is FETCH → ENRICH → TRIAGE. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Severity accuracy check

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded alert `data-exfiltration-03` (whose mock-paired `EnrichedAlert` has `derivedSeverity = CRITICAL` from a `HIGH`-confidence `T1041` pattern).
2. Wait for `EVALUATED`.

**Expected:**
- The recorded `TriageTicket.severity` equals `CRITICAL` (matching `EnrichedAlert.derivedSeverity`).
- The eval score chip shows 5/5 with rationale "all checks passed".
- If the agent's first iteration produced `severity = HIGH` (copying from `AlertDetail.severity` rather than `EnrichedAlert.derivedSeverity`), the evaluator would have flagged it with a "severity accuracy" failure and a score of 4. The presence of a score of 5 confirms the accuracy rule held.

## J6 — Alert with no matching indicator produces empty-pattern ticket

**Preconditions:** Service running. Any model provider. A seeded alert whose raw fields contain no indicator recognised by `EnrichTools.lookupMitreTechnique`.

**Steps:**
1. In the App UI, pick the seeded alert `unknown-traffic-05` (which has raw fields that do not match any entry in the bundled ATT\&CK indicator lookup).
2. Click **Run pipeline**.

**Expected:**
- `EnrichTools.lookupMitreTechnique` returns empty lists for every field.
- The agent's ENRICH task returns an `EnrichedAlert` with `attackPatterns = []` and `derivedSeverity` copied from the raw `AlertDetail.severity`.
- The workflow advances to `ticketStep`. The TRIAGE task returns a `TriageTicket` with `title = "(no ATT&CK mapping)"` and a one-sentence `summary` explaining the gap.
- The eval score chip shows 1 (ATT&CK technique presence fails — no patterns; the other three rules cannot be evaluated). The rationale names "no attack patterns to score."
- The pipeline completes; nothing crashes; the ticket is honestly empty.
