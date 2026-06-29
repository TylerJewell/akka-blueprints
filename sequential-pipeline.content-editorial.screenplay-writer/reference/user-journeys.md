# User journeys — screenplay-writer

Acceptance tests as numbered journeys. Each must pass for the blueprint to be considered correctly generated.

## J1 — Thread to screenplay

**Preconditions:** service running on `http://localhost:9412/`; a model provider key set, or the mock provider selected.

**Steps:**
1. In the App UI tab, paste a multi-message thread and submit.
2. Observe the job list.

**Expected:**
- The POST returns a `jobId`; a row appears in `RECEIVED`.
- The row advances `SANITIZED → ANALYZED → DIALOGUE_WRITTEN → COMPLETED` within ~60 s.
- The detail panel shows a non-empty `screenplay`, a non-empty `characters` list, and a `redactionCount` integer.

## J2 — Redaction before agents

**Preconditions:** as J1.

**Steps:**
1. Submit a thread that contains a real name and an email address (for example `From: Dana Reed <dana.reed@example.com>`).
2. Inspect the job at the `SANITIZED` transition via `GET /api/jobs/{jobId}`.

**Expected:**
- `redactionCount >= 1`.
- `sanitizedText` contains no raw email address or the original name — they are replaced by redaction tokens.
- `characters` labels are descriptive, never a redacted token.

## J3 — Leak blocked

**Preconditions:** as J1; use a thread crafted so the model is likely to echo a source address into the output (or, with the mock provider, the screenplay-formatter mock entry that returns `containsPii = true`).

**Steps:**
1. Submit the thread.
2. Watch the job lifecycle.

**Expected:**
- The job reaches `DIALOGUE_WRITTEN`, then `BLOCKED` — never `COMPLETED`.
- `piiFindings` is non-empty and renders as a red notice in the detail panel.
- No `ScreenplayCompleted` event is recorded for that job.

## J4 — Background simulation

**Preconditions:** service running; no UI interaction.

**Steps:**
1. Wait ~60 s after startup without submitting anything.

**Expected:**
- `ThreadSimulator` enqueues threads from `sample-events/threads.jsonl` every 30 s.
- `ThreadIntakeConsumer` starts one workflow per queued thread; each appears in the job list and reaches `COMPLETED` (or `BLOCKED` for a thread that trips the guardrail).

## J5 — Metadata tabs render

**Preconditions:** service running.

**Steps:**
1. Open the Eval Matrix tab, then the Risk Survey tab.

**Expected:**
- Eval Matrix shows two controls (S1 sanitizer green pill, G1 guardrail red pill) in `matrix-card` style.
- Risk Survey shows pre-filled answers, with deployer-specific fields muted as "To be completed by deployer".
