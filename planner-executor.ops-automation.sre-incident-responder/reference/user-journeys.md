# User journeys — sre-incident-responder

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path: alert triaged, investigated, approved, mitigated, scored

**Preconditions:** Service running on port 9674; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9674/`. App UI tab is visible.
2. In the alert form, type "CPU spike on payment-service: p99 latency 4.2 s, error rate 12%". Select severity `HIGH`. Click Submit.
3. A new incident card appears with status `TRIAGING`.

**Expected:**
- Within 5 s, status transitions to `INVESTIGATING` via SSE.
- The probe log shows at least 3 entries spanning at least two different `ProbeKind` values (e.g., one `METRICS` entry, one `LOGS` entry, one `RUNBOOK` entry).
- Status transitions to `AWAITING_APPROVAL`. The approval pane renders the proposed `RemediationAction` with all fields visible.
- SRE clicks `Accept`. Status transitions to `REMEDIATING`.
- Within ~5 minutes total from submission, status transitions to `MITIGATED`.
- The expanded view shows:
  - An investigation ledger with a non-empty `probePlan` (3–8 steps) and `currentProbe = null` at completion.
  - A remediation ledger with at least one `ActionOutcome` with `succeeded = true`.
  - A `PostIncidentReport` with an 80–140 word summary, a non-empty `rootCauseDiagnosis`, and at least 2 `lessonsLearned` bullets.
  - An `EvalScore` with numeric scores for `investigationCoverageScore` and `hypothesisAccuracyScore`, and at least 2 `findings` bullets.

## J2 — Guardrail blocks an out-of-policy remediation proposal

**Preconditions:** As J1.

**Steps:**
1. Submit the prompt "Database queries timing out on order-service: all reads returning 30 s timeouts." Select severity `CRITICAL`.

**Expected:**
- The commander's `PROPOSE_REMEDIATION` step produces a `RemediationAction` with `estimatedImpact = CRITICAL`.
- The `RemediationGuardrail` rejects it; a `ProbeBlocked` entry appears with `verdict = BLOCKED_BY_GUARDRAIL` and a blocker reason indicating the CRITICAL-impact restriction.
- The commander revises the proposal to an action with `estimatedImpact` of `MEDIUM` or lower (e.g., `RESTART_SERVICE` on the DB proxy).
- The revised proposal either completes the journey to `MITIGATED` or exhausts the replan budget and ends in `UNRESOLVED`. In either case, the original CRITICAL-impact action is never sent to `RemediationAgent`.

## J3 — SRE rejects the approval; incident replans and resolves (or fails)

**Preconditions:** As J1.

**Steps:**
1. Submit "Memory pressure on auth-service: OOM kills every 15 minutes." Select severity `HIGH`.
2. Wait for the approval pane to appear with a proposed `RemediationAction`.
3. Click `Reject`. Enter reason: "Wrong target service — should be auth-worker, not auth-service."
4. Observe the workflow.

**Expected:**
- `ApprovalRejected` event fires on `IncidentEntity`. Status returns to `INVESTIGATING`.
- The probe log gains additional entries as the commander replans using the rejection reason.
- The commander either:
  - Proposes a corrected action targeting `auth-worker`, which appears in a new approval pane. If accepted, the incident reaches `MITIGATED`.
  - Exhausts the replan budget and the incident ends in `UNRESOLVED` with a clear `failureReason`.
- In no case does the rejected action execute on `RemediationAgent`.

## J4 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any alert. Wait for status `INVESTIGATING`.
2. While a probe is in flight (within the first ~10 s), click **Halt new probes** in the operator pane and enter a reason.
3. Observe the in-flight probe completes.

**Expected:**
- The in-flight `ProbeEntry` is recorded normally.
- The next loop iteration reads the halt flag, exits the loop, and emits `IncidentHaltedOperator`.
- Incident status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- Other incidents already queued do not start their workflows until the operator clicks **Resume**.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.

## J5 — Post-execution safety halt fires on unsafe action outcome

**Preconditions:** As J1, with `model-provider = mock` so the `remediation.json` fixture containing a `ROLLBACK_DEPLOYMENT` outcome with "cascade" in `observedEffect` is reachable.

**Steps:**
1. Submit "Deployment regression on checkout-service: error rate jumped to 28% after v2.4.1 rollout." Select severity `HIGH`.
2. Accept the proposed `ROLLBACK_DEPLOYMENT` action in the approval pane.

**Expected:**
- `RemediationAgent` returns an `ActionOutcome` whose `observedEffect` contains the word "cascade".
- `SafetyEvaluator.evaluate(outcome)` flags the result as `UNSAFE`.
- The workflow transitions to `haltedStep` and emits `IncidentHaltedAutomatic`.
- Incident status moves to `HALTED`. `haltReason` identifies the safety evaluator's finding.
- No further remediation actions are attempted.

## J6 — Stale incident times out

**Preconditions:** As J1, with `application.conf` test override `stale.threshold-minutes = 1`.

**Steps:**
1. Submit an alert and configure the mock LLM to return only `ReplanInvestigation` responses so no progress is made.

**Expected:**
- After 1 minute of `INVESTIGATING` without progress, `StaleIncidentMonitor` calls `IncidentEntity.timeoutIncident`.
- The workflow's next `investigateDecideStep` reads `status = TIMED_OUT` and exits via `timedOutStep`.
- Incident status moves to `TIMED_OUT`. `failureReason` is `"stale: no progress after 1m"`.
