# User journeys — cyber-guardian-agent

## J1 — CRITICAL signal triggers automatic halt and playbook

**Preconditions:** Service running on declared port; valid model-provider API key set or mock LLM selected; `TelemetryPoller` enabled.

**Steps:**
1. Open `http://localhost:9181/` → App UI tab.
2. Wait up to 10 s for the first simulated signal.
3. Observe an incident card with severity pill CRITICAL appear in the list.

**Expected:**
- Within 20 s of arrival, the incident status reaches HALTED. The right panel shows the halt directive box with `isolatedHost` and `justification` populated.
- The playbook steps list appears below the halt box with 4–7 numbered steps.
- No network isolation actually occurs (the blueprint simulates in-process); a `HaltIssued` event is present in `GET /api/threats/{id}`.
- The Lift Halt button is visible and enabled.

## J2 — Operator lifts a halt; incident closes; eval event emitted

**Preconditions:** An incident in HALTED status.

**Steps:**
1. Select the HALTED incident.
2. Click Lift Halt. A reason textarea opens.
3. Type a reason ("Confirmed false positive — scanner test"); click Confirm.

**Expected:**
- Status transitions to REMEDIATION immediately. The halt directive box shows the `liftDecision` (liftedBy, reason, liftedAt).
- Within 1 s the status transitions to CLOSED (the workflow's `closeStep` fires).
- Within the next `EvalEventEmitter` tick (up to 5 min), the incident card shows an eval event chip with `haltIssued=true`, `playbookGenerated=true`.

## J3 — LOW severity signal closes without halt or playbook

**Preconditions:** Service running; simulator has at least one LOW-severity signal in the feed.

**Steps:**
1. Wait for a LOW-severity signal to appear in the list.

**Expected:**
- The incident progresses RECEIVED → CLASSIFIED → IGNORED without entering HALTED or REMEDIATION.
- No halt directive is present; no playbook is present.
- The status indicator shows IGNORED styling (muted).
- Within the next `EvalEventEmitter` tick, an eval event is emitted with `haltIssued=false`, `playbookGenerated=false`.

## J4 — Eval event record appears for all closed incidents

**Preconditions:** At least one CLOSED or IGNORED incident exists with no eval event. `EvalEventEmitter` schedule not modified (5-minute default).

**Steps:**
1. Approve a lift (J2) or wait for a LOW/MEDIUM signal to close (J3).
2. Wait up to 5 minutes.

**Expected:**
- `GET /api/threats/{id}` for the closed incident returns `evalEvent` populated with `severity`, `category`, `haltIssued`, `playbookGenerated`, `confidenceScore`, and `closedAt`.
- The App UI incident card shows an eval event chip row.

## J5 — High-throughput simulation does not drop incidents

**Preconditions:** Service running. Set `TELEMETRY_POLLER_INTERVAL_MS=2000` to drip signals every 2 s.

**Steps:**
1. Let the service run for 2 minutes (approximately 60 signals).
2. Call `GET /api/threats?status=CLASSIFIED` and `GET /api/threats?status=HALTED`.

**Expected:**
- Every signal in `threat-signals.jsonl` has a corresponding `IncidentEntity` — no signals are silently dropped.
- CRITICAL signals all have a `halt` field populated; HIGH signals have a `playbook` field; MEDIUM/LOW do not.
- No workflow is in a stuck RECEIVED state after 2 minutes.
