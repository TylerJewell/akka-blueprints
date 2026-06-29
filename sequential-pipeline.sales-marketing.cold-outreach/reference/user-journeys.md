# User journeys — cold-outreach

## J1 — Submit a prospect and send an approved email

**Preconditions:** Service running on declared port (`http://localhost:9969/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded prospect `Acme DevTools <dev@acme.example>` has a matching `src/main/resources/sample-data/prospects/acme-devtools.json` file.

**Steps:**
1. Open `http://localhost:9969/` → App UI tab.
2. From the **Pick a seeded prospect** dropdown, pick `Acme DevTools`.
3. Click **Run outreach**.
4. When the card reaches `AWAITING_REVIEW` and the yellow banner appears, read the draft in the right pane and click **Approve** with note "Looks good."

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `RESEARCHING` within 1 s more.
- Within ~20 s the card reaches `RESEARCHED`. The right pane shows the Research panel with firmographic data and ≥ 1 intent signal.
- Within ~20 s more the card reaches `DRAFTED`. The right pane shows the Email draft panel with a subject line, a body containing `[unsubscribe]` and `[sender-address]` sentinels, and a green `PASSED` compliance badge.
- The card transitions to `AWAITING_REVIEW` and the yellow "Awaiting reviewer approval" banner appears with Approve / Reject buttons.
- After the reviewer clicks Approve, the card transitions to `SENDING` within 1 s, then `SENT` within ~5 s. The right pane shows the Sent email panel with a `messageId`, recipient, and sent timestamp.
- Total elapsed time (excluding reviewer think-time): ≤ 60 s.

## J2 — Send-gate guardrail blocks a premature sendEmail call

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `draft-email.json` includes one entry whose `tool_calls` array contains a `sendEmail` call during the DRAFT phase — before any review decision is recorded.

**Steps:**
1. Submit any seeded prospect until the deliberately-violating mock entry fires (the mock selects it on the first iteration of every 3rd prospect by `seedFor(prospectId)` modulo).
2. Watch the prospect's lifecycle in the network panel (`/api/outreach/sse`).

**Expected:**
- On the prospect's `draftStep`, the agent's first iteration calls `sendEmail`. `SendGuardrail` rejects it; a `GuardrailRejected{phase: "DRAFT", tool: "sendEmail", reason: "send-blocked: ..."}` event lands on the entity.
- The `sendEmail` tool body is never reached — no entry appears in `src/main/resources/sent-log/sent.jsonl` for this call.
- The agent's next iteration falls through to a normal DRAFT sequence (`personalizeLine` + `renderTemplate`). The card eventually reaches `AWAITING_REVIEW` as in J1.
- The card in the App UI shows a small red dot. The rejection-log strip on the right pane shows the one rejected call with its structured reason.

## J3 — Compliance guardrail rejects a draft missing the unsubscribe sentinel

**Preconditions:** Mock LLM mode. The mock's `draft-email.json` includes one entry whose body does not contain the `[unsubscribe]` sentinel.

**Steps:**
1. Submit any seeded prospect. (The non-compliant draft entry is selected on a predictable modulo by `seedFor`.)
2. Watch the live list for the compliance badge to briefly show `FAILED` before the agent revises.

**Expected:**
- `ComplianceGuardrail` rejects the agent's proposed response; a `compliance-violation: unsubscribe-missing` rejection is returned.
- The agent's next iteration produces a revised draft that includes `[unsubscribe]`. The revised draft passes the compliance check.
- The card's compliance badge shows `PASSED` after the revision. The compliance-check audit event is visible in the entity log (the `ComplianceChecked{passed: false, failedRules: ["unsubscribe-missing"]}` event precedes the `ComplianceChecked{passed: true}` event).

## J4 — Reviewer rejects the draft; no email is sent

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit any seeded prospect and wait for `AWAITING_REVIEW`.
2. In the right pane, click **Reject** and enter the reason "Wrong tone for this account."

**Expected:**
- The card transitions from `AWAITING_REVIEW` to `REVIEW_REJECTED` within 1 s.
- No `EmailSent` event appears in the entity log.
- No entry is appended to `src/main/resources/sent-log/sent.jsonl`.
- The right pane shows the rejection reason "Wrong tone for this account." and the reviewer's timestamp.
- The `sendEmail` tool is never called — no `debug:agent.tool.call` line for `sendEmail` appears in the service log for this prospect.

## J5 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider (real or mock — the guardrails run either way).

**Steps:**
1. Submit any seeded prospect.
2. Approve the draft when `AWAITING_REVIEW` is reached.
3. Wait for `SENT`.
4. Inspect the service log for tool-call lines tagged with the `prospectId`.

**Expected:**
- The RESEARCH task's log entries show only `lookupFirmographics` and `fetchIntentSignals` calls.
- The DRAFT task's log entries show only `personalizeLine` and `renderTemplate` calls.
- The SEND task's log entries show only `sendEmail` calls.
- No cross-phase calls appear. If a guardrail violation path fires, a `guardrail.reject` line precedes the agent's retry; the rejected call is logged but never executed.
- The order of tasks in the log is RESEARCH → DRAFT → SEND. Between DRAFT and SEND the log shows a `workflow.step.pause` line for `approvalStep` and a `workflow.step.resume` line after the review decision.

## J6 — Custom prospect with no matching firmographic data

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/prospects/<slug>.json` exists for the custom company.

**Steps:**
1. In the App UI, enter `newco@unknown.example` and company name `Unknown Startup Co.` into the inputs.
2. Click **Run outreach**.

**Expected:**
- `ResearchTools.lookupFirmographics` returns an empty `FirmographicData` with `intentSignals = []`.
- The agent's RESEARCH task returns a `ProspectProfile` with `personalizationHooks = []`.
- The workflow advances to `draftStep`. The agent produces a `DRAFT` that references only `companyName` and `industry` (no personalization hooks), per the refusal instruction in `prompts/outreach-agent.md`.
- The draft contains `[unsubscribe]` and `[sender-address]` — compliance passes.
- The pipeline reaches `AWAITING_REVIEW` with a valid draft. The reviewer can approve and the pipeline sends. Nothing crashes; the generic draft is honestly generic.
