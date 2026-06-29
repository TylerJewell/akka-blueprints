# User journeys — campaign-optimizer-loop

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path: brief → approval → live → completed

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:<port>/`. App UI tab is visible.
2. In the campaign form, enter goal "Run a product-launch email campaign targeting enterprise buyers in North America", target audience "Enterprise decision-makers, company size > 500", channels "email". Click Submit.
3. A new campaign card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- The run ledger shows at least one COPY step and one AUDIENCE step, both with `verdict = OK`.
- Status transitions to AWAITING_APPROVAL. An approval card appears in the approval queue pane.
- A user clicks `Approve` with `decidedBy = "demo@example.com"`. Status transitions to LIVE.
- Within ~4 minutes of initial submission, `PerformanceMonitor` reads a KPI fixture with all metrics above threshold and the Planner emits `CampaignCompleted`. Status transitions to COMPLETED.
- The expanded view shows a `CampaignReport` with a 60–120 word summary and 3–5 highlight bullets.

## J2 — Guardrail blocks a competitor brand mention in copy

**Preconditions:** As J1, with the `copy-writer.json` mock containing one fixture entry whose output includes a competitor brand name (e.g., "AcmeCorp").

**Steps:**
1. Submit any brief whose copy step maps to the fixture entry containing the competitor mention.

**Expected:**
- The `CopyWriterAgent` returns a `StepResult` whose `output` contains the competitor brand name.
- `CopyGuardrail.check` rejects it; a `StepBlocked` entry appears in the run ledger with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker = "COPY_VIOLATION:competitor-brand-mention"`.
- The Planner sees the blocker and proposes a revised COPY step. The revised step either produces compliant copy and the campaign proceeds, OR the Planner exhausts the replan budget and the campaign ends in `FAILED` with a clear `failureReason`. Either outcome is acceptable; the non-compliant copy must never reach PUBLISH or AWAITING_APPROVAL.
- The approval queue pane never shows this campaign while the copy violation is unresolved.

## J3 — Marketer rejects the approval request

**Preconditions:** As J1. Campaign has reached AWAITING_APPROVAL.

**Steps:**
1. In the approval queue pane, click `Reject` on the pending campaign card.
2. Fill in `decidedBy = "maria@example.com"` and `note = "Copy tone does not match brand guidelines for regulated markets."` Click Reject.

**Expected:**
- `ApprovalEntity` records `ApprovalDenied`.
- `CampaignWorkflow`'s `approvalGateStep` unblocks with a denial, emits `CampaignRejected`.
- Campaign status moves to REJECTED. `rejectionReason` is populated with the note.
- No `AssetPublisher` steps appear in the run ledger — `PUBLISH` specialist was never dispatched.
- The campaign card's left border turns orange in the live list. The expanded view shows the rejection reason paragraph.

## J4 — Performance monitor triggers re-optimization

**Preconditions:** As J1, campaign has reached LIVE. KPI fixture for this campaign has `openRate = 0.03` (below threshold 0.08).

**Steps:**
1. Allow the `PerformanceMonitor` to tick (within 60 s of the campaign going LIVE).

**Expected:**
- `PerformanceMonitor` reads the KPI fixture and detects `openRate 0.03 < threshold 0.08`.
- `CampaignEntity.firePerformanceAlert` is called; `PerformanceAlertFired` is emitted.
- `CampaignWorkflow` re-enters the executor loop; `PerformanceAnalystAgent` is dispatched and returns a `PerformanceAssessment` with `kpisMet = false` and recommendations.
- A new `RunEntry` appears in the run ledger with `specialist = PERFORMANCE` and `verdict = KPI_MISS`, and a `MetricsSnapshot` showing `openRate = 0.03`.
- The Planner revises the ledger and dispatches a new COPY step with a revised subject line.
- The re-optimized copy passes the guardrail, a new PUBLISH step runs, and the campaign remains in LIVE status awaiting the next KPI tick.
- The expanded run ledger shows the full re-optimization sequence in timeline order.
