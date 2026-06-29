# User Journeys — Lead Score HITL

Acceptance journeys. Each generated system must pass these.

## J1 — Submit a lead and watch it score

**Preconditions:** service running on `http://localhost:9178`; a model provider configured (or mock).
**Steps:**
1. `POST /api/leads` with `{ "company": "Acme Robotics", "contactName": "Jordan Lee", "rawSource": "trade show referral" }`.
2. Subscribe to `/api/leads/sse`.
**Expected:** the response returns a `leadId`. Over SSE the lead moves `NEW` → `COLLECTED` → `ANALYZED` → `SCORED` within ~60s, ending with a non-empty `score` (0–100) and `scoreRationale`. The persisted `contactName` is the sanitized form.
**Done when:** the lead row shows `SCORED` with a score and rationale.

## J2 — Top lead gets shortlisted

**Preconditions:** at least one `SCORED` lead exists.
**Steps:**
1. Wait for the next `LeadRankingMonitor` tick (≤30s).
**Expected:** the top-ranked `SCORED` leads (up to 3) transition to `SHORTLISTED` and show `shortlisted: true`; Approve/Reject controls appear on those rows in the App UI.
**Done when:** a high-scoring lead reads `SHORTLISTED` with review controls visible.

## J3 — Approve a shortlisted lead and draft outreach

**Preconditions:** a `SHORTLISTED` lead.
**Steps:**
1. Click Approve (or `POST /api/leads/{id}/approve` with `{ "reviewedBy": "demo", "comment": "go" }`).
**Expected:** the lead moves to `APPROVED`, the workflow resumes, `OutreachAgent` drafts an email under the before-response guardrail, and the lead reaches `CONTACTED` with non-empty `outreachSubject` and `outreachBody` within ~60s.
**Done when:** the lead reads `CONTACTED` and the row expands to show the drafted subject and body.

## J4 — Reject a shortlisted lead

**Preconditions:** a `SHORTLISTED` lead.
**Steps:**
1. Click Reject (or `POST /api/leads/{id}/reject` with `{ "reviewedBy": "demo", "reason": "out of ICP" }`).
**Expected:** the lead moves to terminal `REJECTED` with `reviewComment` set; no outreach is drafted and `outreachSubject`/`outreachBody` stay null.
**Done when:** the lead reads `REJECTED` with the reason shown and no outreach fields.

## J5 — Background load from the simulator

**Preconditions:** service running, no UI interaction.
**Steps:**
1. Wait through two `LeadSimulator` ticks (~60s).
**Expected:** new leads seeded from `sample-events/inbound-leads.jsonl` appear and progress to `SCORED` without any manual submission.
**Done when:** at least two simulator-seeded leads reach `SCORED`.
