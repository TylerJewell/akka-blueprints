# User journeys — Fair CV Matcher

Acceptance tests as numbered journeys. Each passing means the blueprint generated correctly.

## J1 — Submit a CV and watch it match

**Preconditions:** service running on `http://localhost:9736`; a model provider configured (or mock).
**Steps:**
1. Open the App UI tab. Paste CV text, add two or three postings, set a slice label, click Submit.
2. Observe the candidate appear in `SUBMITTED` over SSE.
**Expected:** within ~30s the candidate advances `EXTRACTED → SANITIZED → MATCHED`, and the `matches` list shows one scored result per posting, ranked descending. `GET /api/candidates/{id}` returns `status` `MATCHED` with non-empty `matches`.

## J2 — Sanitization holds

**Preconditions:** a candidate has reached `MATCHED`.
**Steps:**
1. Expand the candidate's detail in the App UI.
2. Inspect the redaction summary and the matching rationale text.
**Expected:** `redactions` is non-empty (name/address/age/gender as available), and no match `rationale` references any protected attribute. The matching agent's input carried no name/address/age tokens.

## J3 — Reviewer records oversight

**Preconditions:** a candidate in `MATCHED`.
**Steps:**
1. Click Review on the candidate, enter a reviewer id and a note, submit.
**Expected:** `POST /api/candidates/{id}/review` returns 200; the candidate moves to `REVIEWED` with `reviewedBy` and `reviewNote` set; the SSE stream pushes the updated row.

## J4 — Fairness flag

**Preconditions:** the simulator has run long enough to populate slices, or the sample CVs skew one slice.
**Steps:**
1. Open the fairness strip in the App UI, or call `GET /api/fairness`.
**Expected:** each slice shows a selection rate and parity ratio; a slice whose ratio is below 0.8 is reported with `flagged: true`.

## J5 — Background load from the simulator

**Preconditions:** service running, no UI interaction.
**Steps:**
1. Leave the service idle for a minute.
**Expected:** `CvSimulator` enqueues a fresh CV every 30s; new candidates appear and run to `MATCHED` without any user submission.
