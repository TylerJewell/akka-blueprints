# User journeys — ad-spend-payment-agent

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path: brief → copy → approval → payment → completed

**Preconditions:** Service running on port 9826; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9826/`. App UI tab is visible.
2. Fill in the brief form: Title = "AkkaStore Developer Banner Ad", Brand Voice = "direct, technical", Target Audience = "JVM developers", Ad Format = "banner-728x90", Budget = 0.05 ETH. Click Submit.
3. A new campaign card appears with status PLANNING.
4. Within ~30 s, the status transitions to EXECUTING via SSE. The creative ledger shows a non-empty plan.
5. After one or two COPYWRITER steps complete, the status transitions to AWAITING_APPROVAL. The pending-approvals pane shows one row for the placement.
6. Click Approve and enter an approver name. The pane row updates to APPROVED via SSE.
7. The workflow resumes; status returns to EXECUTING, then COMPLETED.

**Expected:**
- The final report shows at least one `PlacementEntry` with `approvalStatus = APPROVED` and `paymentResult.ok = true` with a non-empty `txHash`.
- `totalSpentWei` in the report is greater than zero and less than or equal to `budgetWei`.
- The creative ledger shows 2–4 entries; the payment ledger shows the completed placement.

## J2 — Guardrail blocks an over-budget payment dispatch

**Preconditions:** As J1.

**Steps:**
1. Submit a brief with Budget = 0.001 ETH (1 000 000 000 000 000 wei).
2. The planner proposes a PAYMENT dispatch with `proposedSpendWei = 2 000 000 000 000 000` (0.002 ETH).

**Expected:**
- The guardrail step blocks the dispatch; a `CreativeEntry` with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker = "proposedSpendWei exceeds remaining budget"` appears in the creative ledger.
- The planner either replans with a lower-spend placement (and that placement's `proposedSpendWei` is within budget) or exhausts the replan budget and the campaign ends in `FAILED` with a clear `failureReason`.
- In neither case does a payment call reach `PaymentExecutorAgent`.

## J3 — Operator halt drains gracefully; no payment fires after halt

**Preconditions:** As J1.

**Steps:**
1. Submit any brief.
2. While the campaign is EXECUTING (during a COPYWRITER step), click **Halt new dispatches** in the operator pane and provide a reason.
3. Observe the in-flight copywriter task completes and its result is recorded.

**Expected:**
- The in-flight `CreativeEntry` is recorded normally.
- The next loop iteration reads the halt flag, exits the loop, and emits `CampaignHaltedOperator`.
- Campaign status moves to `HALTED`. `haltReason` is populated with the operator's text.
- No `PaymentApprovalRequested` event fires after the halt was set.
- The operator pane shows the `HALTED` pill in real time via the `control-update` SSE event.
- Clicking Resume clears the flag; subsequent new brief submissions start normally (this campaign stays HALTED).

## J4 — Credential sanitizer scrubs a wallet private key from creative output

**Preconditions:** As J1, with the canned mock file `mock-responses/copywriter.json` present and containing one entry whose `content` field includes a bare 64-character hex string (the fixture for the sanitizer test).

**Steps:**
1. Submit any brief. The BriefSimulator or manual submission triggers the copywriter mock path that includes the hex key in its response.

**Expected:**
- The `CredentialScrubber` replaces the 64-char hex string with `[REDACTED:wallet-private-key]` before the entry is recorded.
- The `CreativeEntry.sanitizedContent` in the payment ledger does NOT contain the raw hex string.
- The planner's next prompt (visible in subsequent `DECIDE_NEXT` calls) sees only the redacted form.
- The final `CampaignReport.summary` does not contain the raw hex string.
- The UI's expanded creative ledger renders the redacted span in italics with a tooltip showing `wallet-private-key`.

## J5 — Approval gate timeout marks campaign STUCK

**Preconditions:** As J1, with `application.conf` test override `stuck.threshold-minutes = 2`.

**Steps:**
1. Submit a brief.
2. The campaign reaches AWAITING_APPROVAL.
3. Do not click Approve or Reject. Wait 2 minutes.

**Expected:**
- `StaleCampaignMonitor` detects the campaign stuck in `AWAITING_APPROVAL` past the threshold and calls `CampaignEntity.timeoutFail`.
- Campaign status moves to `STUCK`. `failureReason` is `"stuck: no approval action after 2m"`.
- The pending-approvals pane row for the placement remains visible but the campaign card shows the `STUCK` status pill.
