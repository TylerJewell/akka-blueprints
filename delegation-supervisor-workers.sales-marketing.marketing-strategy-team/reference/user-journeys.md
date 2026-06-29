# User journeys

Acceptance journeys. Passing all of these means the blueprint generated correctly.

## J1 — Submit a brief, watch a plan assemble

**Preconditions:** service running on `http://localhost:9788/`; a model provider configured (or mock).
**Steps:**
1. Open the App UI tab.
2. Type a project brief (e.g. "Launch a B2B time-tracking app for remote design teams; goal: 500 trial signups").
3. Click Submit.
**Expected:** the response carries a `campaignId`; the campaign appears in `RECEIVED`, then advances through `RESEARCHED → STRATEGIZED → CONTENT_DRAFTED → ASSEMBLED → EVALUATED` over the SSE stream within ~60 s. The `ASSEMBLED` card shows a non-empty plan summary; the `EVALUATED` card shows a score.

## J2 — Grounded claims only

**Preconditions:** a campaign reaching the assemble step.
**Steps:**
1. Submit a brief and let it run to `ASSEMBLED`.
2. Inspect the assembled plan's claims against the recorded `MarketFindings`.
**Expected:** every factual claim in the plan summary is present in `findings.claims`. If the lead agent's first assemble produces an ungrounded claim, the G1 guardrail fails the assemble step and it re-runs (visible as a brief stall before `ASSEMBLED`); the persisted plan has only grounded claims.

## J3 — Self-eval score

**Preconditions:** a campaign reaching `EVALUATED`.
**Steps:**
1. Open the `EVALUATED` campaign card.
**Expected:** `evalScore` is an integer 0–100 and `evalNotes` references the brief's goal. Both render on the card.

## J4 — Background load from the simulator

**Preconditions:** service running; no user interaction.
**Steps:**
1. Leave the App UI tab open without submitting.
**Expected:** within ~45 s the `BriefSimulator` enqueues a canned brief; `BriefConsumer` starts a workflow; a campaign runs end-to-end to `EVALUATED` on its own.

## J5 — Metadata tabs render

**Preconditions:** service running.
**Steps:**
1. Open the Risk Survey tab, then the Eval Matrix tab.
**Expected:** Risk Survey renders the pre-filled answers in `matrix-card` style with deployer placeholders muted; Eval Matrix renders two controls (G1 guardrail, E1 eval-event) each with a colored mechanism pill, expandable on click.
