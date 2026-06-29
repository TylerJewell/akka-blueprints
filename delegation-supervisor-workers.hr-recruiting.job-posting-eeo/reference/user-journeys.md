# User journeys — job-posting-eeo

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on declared port; a model-provider key set, or the mock LLM selected during scaffolding.

**Steps:**
1. Open `http://localhost:9795/`. App UI tab is visible.
2. In the Company field type "Northwind Logistics"; in the Role title field type "Warehouse Operations Lead". Click Submit.
3. A new posting card appears with status PLANNING.

**Expected:**
- Within 1 s, status transitions to ANALYZING via SSE.
- Within 60 s, status transitions through DRAFTED → SANITIZED → CLEARED.
- The expanded view shows: a culture profile (3–5 values, a tone, a summary), a role spec (4–6 responsibilities, 4–6 qualifications, a market salary range), a draft body of 120–200 words, an EEO statement, and an empty `removedTerms` list.

## J2 — Guardrail block

**Preconditions:** As J1, plus a test request whose draft is forced to contain discriminatory language (e.g., the topic "young energetic team only" or a test-mode override the deterministic matcher rejects).

**Steps:**
1. Submit the flagged request.

**Expected:**
- Posting progresses PLANNING → ANALYZING → BLOCKED.
- The expanded view shows the draft populated but with `guardrailVerdict: "blocked: <reason>"` and `failureReason` set. The culture and role analyses remain visible for audit.

## J3 — Sanitizer strips a protected-class preference

**Preconditions:** As J1, plus a draft that contains a protected-class preference term the sanitizer's term list matches.

**Steps:**
1. Submit a request that produces such a draft.

**Expected:**
- Posting reaches SANITIZED then CLEARED.
- The sanitized body no longer contains the term; `removedTerms` is non-empty and the removed terms render as chips in the App UI.
- The raw `draft.body` (pre-sanitizing) is still retained on the entity for audit.

## J4 — Degraded worker

**Preconditions:** As J1, plus the `RoleAnalyst` step timeout reduced to 1 s (`ROLE_TIMEOUT_MS=1000` or by editing `application.conf`).

**Steps:**
1. Submit any request.

**Expected:**
- Posting progresses PLANNING → ANALYZING → DEGRADED.
- The expanded view shows the CultureAnalyst output populated, the role spec empty, the draft body acknowledging the missing role analysis, and `failureReason: "worker timeout: RoleAnalyst"`.
