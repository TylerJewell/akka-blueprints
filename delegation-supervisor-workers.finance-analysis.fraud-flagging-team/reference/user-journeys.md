# User journeys — fraud-flagging-team

Acceptance tests as numbered journeys. Each has preconditions, steps, and expected outcome. Passing J1–J4 means the blueprint generated correctly.

## J1 — Submit a transaction and watch it analyze

**Preconditions:** service running on `http://localhost:9778/`; a model provider key set or the mock provider selected.

**Steps:**
1. On the App UI tab, submit a transaction (`customerId`, `amount`, `transactionRef`, `memo`).
2. Observe the case list via SSE.

**Expected:** a case appears in `ANALYZING`. Within ~30s the three worker findings (`fraudScore`, `complianceNotes`, `riskTier`) and the `supervisorRationale` populate, and the case reaches `FLAGGED` or `CLEARED`.

## J2 — Confirm a flagged case

**Preconditions:** a case in `FLAGGED` (from J1 or the feed).

**Steps:**
1. Click Confirm on the flagged case.

**Expected:** the case moves to `CONFIRMED`, the before-tool-call guardrail passes, the gated account action runs, and the case reaches `ACTIONED` with a non-null `actionTaken` within ~30s.

## J3 — Dismiss a flagged case

**Preconditions:** a case in `FLAGGED`.

**Steps:**
1. Click Dismiss, enter a note, submit.

**Expected:** the case reaches `DISMISSED` with the `analystNote` set. No account action runs; `actionTaken` stays null.

## J4 — Auto-escalation

**Preconditions:** a case in `FLAGGED` left untouched.

**Steps:**
1. Wait past the review window without acting.

**Expected:** `StuckCaseMonitor` transitions the case to `ESCALATED`; the Confirm/Dismiss buttons disappear.

## J5 — Background load from the feed

**Preconditions:** service running, no UI interaction.

**Steps:**
1. Wait for `TransactionSimulator` to drip a transaction from `transactions.jsonl`.

**Expected:** `TransactionConsumer` opens a case and starts a workflow per transaction; cases appear in the list without any manual submit.

## J6 — Redaction in logs

**Preconditions:** service running.

**Steps:**
1. Submit a transaction whose memo and reference contain identifier-like strings.
2. Inspect the service logs.

**Expected:** card/account/customer identifiers are masked in any logged transaction field (control S1); the case JSON over the API still carries the values the UI needs.
