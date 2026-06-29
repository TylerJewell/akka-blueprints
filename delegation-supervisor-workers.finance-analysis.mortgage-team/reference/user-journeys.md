# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit an application and watch parallel assessment

**Preconditions:** service running on `http://localhost:9824/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Fill in the application form (applicantRef, loan product, amounts, income, credit score, document IDs) and Submit.
2. Observe the new application row via SSE.

**Expected:** the application progresses `RECEIVED → UNDER_REVIEW → PENDING_SIGNOFF` within ~90 s. The expanded row shows an `EligibilityAssessment`, a `DocumentationAssessment`, and an `UnderwritingAssessment` populated close together — evidence of parallel execution. The HITL decision panel is visible on the row.

## J2 — Human underwriter approves and application reaches APPROVED

**Preconditions:** an application in `PENDING_SIGNOFF` status (from J1).

**Steps:**
1. Expand the row in the App UI.
2. Fill in the HITL decision panel: select Approve, enter decidedBy and notes.
3. Click Approve.

**Expected:** `POST /api/applications/{id}/decision` with `approved: true` is sent. The application transitions to `APPROVED` via SSE within seconds. CustomerCommsAgent's outcome notification draft is visible in the expanded row. The HITL decision panel disappears.

## J3 — Compliance filter blocks a non-conforming term structure

**Preconditions:** a mock application that causes `UnderwritingAgent` to propose a rate above the configured ceiling (use mock-response overrides or set the product-rules ceiling below the mock's rate).

**Steps:**
1. Submit the fixture application.
2. Watch the application row.

**Expected:** `complianceStep` flags the proposed terms; `ApplicationEntity.applyComplianceHold` is called; the application enters `COMPLIANCE_HOLD` with a `complianceHoldReason`. The HITL decision panel never appears. The application does not advance to `PENDING_SIGNOFF`.

## J4 — Worker timeout routes the application to REVIEW_REQUIRED

**Preconditions:** `UnderwritingAgent` step timeout set to 1 s (test override).

**Steps:**
1. Submit an application.
2. Watch the application row.

**Expected:** `underwritingStep` times out; the workflow routes to `reviewRequiredStep`; the application enters `REVIEW_REQUIRED`. No infinite retry. The underwriter team sees the application flagged in the live list.

## J5 — Eval score appears beside a decided application

**Preconditions:** at least one `APPROVED` or `DECLINED` application without an `evalScore`.

**Steps:**
1. Wait for `DecisionEvalSampler` to run (every 5 minutes), or trigger it.
2. Observe the application row.

**Expected:** the application gains an `evalScore` (1–5) and an `evalRationale` via SSE. The App UI row shows the score. The application's status and the applicant's outcome were never delayed by the eval — it is non-blocking.

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running for 3+ minutes.

**Expected:** `ApplicationSimulator` drips a sample application from `mortgage-applications.jsonl` every 90 s; each goes through the full assessment pipeline. The App UI is non-empty on first load and populates across multiple loan products and credit profiles.
