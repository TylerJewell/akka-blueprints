# User journeys — llm-auditor

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the revision ceiling

**Preconditions:** Service running on port 9503; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9503/`. App UI tab is visible.
2. In the Response text field, paste: "Our team guarantees a 99.9% uptime SLA and you are entitled to a full refund within 30 days." Set channel to "support". Leave max revisions at the default 3. Click Submit.
3. A new session card appears with status `AUDITING`.

**Expected:**
- Within 1 s, the guardrail verdict pill reads `OK` with an `initialSeverity` value below 9.
- Within 60 s of submission, either:
  - Cycle 1's verdict is `PASS` and the session transitions to `APPROVED`, OR
  - Cycle 1's verdict is `REVISE` with 1–4 findings and the session transitions to `REVISING`, with cycle 2 appearing shortly after; this continues until either a `PASS` or the revision ceiling is reached.
- On `APPROVED`, the terminal block shows the approved text and "approved at cycle N."
- The expanded view shows every cycle's input response, guardrail verdict, critic verdict, severity score, findings, and revised response (when produced).

## J2 — Halt at revision ceiling

**Preconditions:** As J1, plus an override that forces the Critic to always return `REVISE` (test mode — submit the literal text `"test-force-escalate"`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Submit the text `"test-force-escalate"` with channel "support" and the default 3 revisions.

**Expected:**
- Session progresses `AUDITING` → `REVISING` → `AUDITING` → `REVISING` → `AUDITING` → `REVISING` for `maxRevisions` cycles (default 3).
- After the 3rd cycle ends in `REVISE`, the session transitions to `ESCALATED` (not stuck in `REVISING`).
- The terminal block shows the lowest-severity cycle's text as "best available" and the `escalationReason` reads `"max revisions reached (3)"`.
- All 3 cycles are present in the expanded view, each with its input, guardrail `OK` verdict, `REVISE` verdict, severity score, and findings.
- `GET /api/sessions/{id}` returns the full Session with all 3 cycles in `cycles[]` and `status: "ESCALATED"`.

## J3 — Guardrail immediate escalation

**Preconditions:** As J1.

**Steps:**
1. Submit a response text containing `"CRITICAL VULNERABILITY EXPLOIT"` (a keyword the severity scanner flags at 9+) with channel "general".

**Expected:**
- The guardrail records `verdict.passed = false`, `reasonCode = "SEVERITY_EXCEEDS_THRESHOLD"`, with the `initialSeverity` value at or above 9.
- No revision cycle is started. `CriticAgent` and `ReviserAgent` are never called.
- The session transitions directly from `AUDITING` to `ESCALATED` within 5 s.
- The expanded view shows cycle 1 with the red `SEVERITY_EXCEEDS_THRESHOLD` guardrail pill, no audit verdict, and no revised response.
- `GET /api/sessions/{id}` returns `status: "ESCALATED"` with an `escalationReason` of `"guardrail blocked: SEVERITY_EXCEEDS_THRESHOLD"` and `cycles[0].verdict = null`.

## J4 — Audit event timeline

**Preconditions:** At least one session has completed (any terminal state).

**Steps:**
1. Click the session card to expand.

**Expected:**
- The timeline shows one `AuditRecorded` event per completed audit cycle, each with `cycleNumber`, `decision`, `severityScore`, and `guardrailBlocked` populated.
- The terminal transition (SessionApproved or SessionEscalated) is also surfaced as a final `AuditRecorded` event carrying the session-level disposition and terminal severity score.
- `GET /api/sessions/{id}` includes an `auditEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
- `AuditSampler` ticks at most 30 s after a cycle completes; the `AuditRecorded` event appears in the UI timeline within 35 s of a cycle finishing.
