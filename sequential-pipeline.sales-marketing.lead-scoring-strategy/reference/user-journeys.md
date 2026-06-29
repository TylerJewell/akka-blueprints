# User Journeys — Lead Scoring Strategy

Acceptance journeys. Each generated system must pass these.

## J1 — Submit a lead and watch it score

**Preconditions:** service running on `http://localhost:9643`; a model provider configured (or mock).
**Steps:**
1. `POST /api/leads` with `{ "company": "Acme Robotics", "contactName": "Jordan Lee", "formResponses": "Need warehouse automation, ~50 robots, budget Q3" }`.
2. Subscribe to `/api/leads/sse`.
**Expected:** the response returns a `leadId`. Over SSE the lead moves `NEW` → `INTAKE` → `RESEARCHED` → `SCORED` within ~90s, ending with a non-empty `score` (0–100) and `scoreRationale`. The persisted `contactName` and `formResponses` are the sanitized forms.
**Done when:** the lead row shows `SCORED` with a score and rationale.

## J2 — Auto-eval fires on the score

**Preconditions:** at least one `SCORED` lead exists.
**Steps:**
1. Watch the same lead over SSE after it reaches `SCORED`.
**Expected:** `ScoreEvalConsumer` records an eval result; the lead gains a non-empty `evalScore` (0–100) and `evalFlags` (`none` or a comma-joined list). The lifecycle status stays `SCORED` until the strategy step runs.
**Done when:** the lead row shows an eval result alongside its score.

## J3 — Strategy produced, lead completes

**Preconditions:** a `SCORED` lead.
**Steps:**
1. Wait for `strategyStep` to run (no manual action needed).
**Expected:** the lead moves to terminal `COMPLETE` with a non-empty `engagementStrategy`. The row expands to show the intake profile, research brief, score rationale, eval result, and strategy.
**Done when:** the lead reads `COMPLETE` and the strategy text is present.

## J4 — Background load from the simulator

**Preconditions:** service running, no UI interaction.
**Steps:**
1. Wait through two `LeadSimulator` ticks (~60s).
**Expected:** new leads seeded from `sample-events/inbound-leads.jsonl` appear and progress to `COMPLETE` without any manual submission.
**Done when:** at least two simulator-seeded leads reach `COMPLETE`.

## J5 — Eval flags a protected-attribute rationale

**Preconditions:** a scored lead whose `scoreRationale` references a protected attribute (age, gender, race, religion, or national origin).
**Steps:**
1. Submit a lead crafted so the model rationale mentions a protected attribute, or inspect a sample lead seeded for this case.
**Expected:** `ScoreEvalConsumer` sets `evalFlags` to a non-`none` value naming the flagged attribute and lowers `evalScore`; the App UI renders the flag in a warning color.
**Done when:** a flagged lead shows a non-`none` `evalFlags` value.
