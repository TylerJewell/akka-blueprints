# User journeys

Acceptance journeys. Each must pass against the generated system.

## J1 — Source and match a candidate

**Preconditions:** service running; not halted; a model provider (real or mock)
configured.
**Steps:**
1. In the App UI, submit a requisition with `forceBlockedDomain` unchecked and
   `candidateCount` = 1.
2. Watch the candidate list via SSE.
**Expected:** the candidate moves SOURCED → SANITIZED → SCREENED → MATCHED within
~60 s. The MATCHED row shows a non-null `matchScore` (0–100) and a `matchRationale`.

## J2 — Sourcing blocked by the domain guardrail

**Preconditions:** service running.
**Steps:**
1. Submit a requisition with `forceBlockedDomain` checked.
**Expected:** the candidate reaches BLOCKED. `blockedDomain` is populated; no
profile fields (`sourceHandle`, screening, scoring) are set. `SourcingBlocked`
was written; the source-candidates tool call never ran against the network.

## J3 — Special-category redaction before scoring

**Preconditions:** service running; at least one candidate has reached SANITIZED.
**Steps:**
1. Submit a requisition and let a candidate progress.
2. Inspect the candidate via `GET /api/candidates/{id}`.
**Expected:** every candidate at SANITIZED or later has a non-empty
`redactedCategories`. The screening summary and match rationale contain no
protected attribute (name, gender, age, ethnicity, photo, marital status).

## J4 — Operator halt and resume

**Preconditions:** service running; not halted.
**Steps:**
1. Click Halt (`POST /api/system/halt`); confirm `/api/system/status` returns
   `{ halted: true }`.
2. Submit a new requisition.
3. Click Resume; submit another requisition.
**Expected:** while halted, the new requisition starts no workflow (no new
candidate progresses past SOURCED; any in-flight candidate that re-checks shows
HALTED). After Resume, a freshly submitted requisition completes the pipeline
normally.

## J5 — Deployer monitoring snapshot

**Preconditions:** several candidates have completed.
**Steps:**
1. Read `GET /api/monitoring`.
**Expected:** `evaluated`, `matched`, `rejected`, and `blocked` counts are
consistent with the candidate list, and `avgScore` is the mean of MATCHED
candidates' scores. The App UI monitoring panel shows the same figures.

## J6 — Background load from the simulator

**Preconditions:** service running; not halted.
**Steps:**
1. Leave the UI idle for ~90 s.
**Expected:** `RequisitionSimulator` enqueues a canned requisition every 45 s;
new candidates appear without any user interaction, at least one of which is
BLOCKED (the canned line with `forceBlockedDomain`).
