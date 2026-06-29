# User journeys

Acceptance journeys. Passing all of these means the blueprint generated correctly.

## J1 — Submit a topic and watch a debate run

- **Preconditions:** service running on `http://localhost:9680/`; a model provider configured (real or mock).
- **Steps:** `POST /api/debates { "topic": "Cities should make public transit free" }`. Subscribe to `/api/debates/sse`.
- **Expected:** the response returns a `debateId`. The debate appears in `DEBATING`, advances through rounds (`currentRound` increments), and reaches `CONCLUDED` with a non-empty `conclusion`.
- **Done when:** the debate row shows `CONCLUDED` and a conclusion in the App UI.

## J2 — Inspect rounds and synthesis

- **Preconditions:** a `CONCLUDED` debate from J1.
- **Steps:** expand the debate row in the App UI.
- **Expected:** every recorded round shows the advocate argument and the critic rebuttal; the key-arguments list has three to five entries drawn from both sides.
- **Done when:** rounds and key arguments are visible and non-empty.

## J3 — Synthesis guardrail blocks an unbalanced conclusion

- **Preconditions:** service running.
- **Steps:** drive a debate whose synthesis fails the balance/safety check (e.g., a mock or model output that references only one side).
- **Expected:** the before-agent-response guardrail blocks the synthesis; the workflow's error step records `DebateFailed`; the debate reaches `FAILED` with a `failedReason` rather than persisting an unbalanced conclusion.
- **Done when:** the debate shows `FAILED` and no one-sided conclusion is persisted.

## J4 — Quality eval surfaces a score

- **Preconditions:** a `CONCLUDED` debate.
- **Steps:** observe the debate after `SynthesisRecorded`.
- **Expected:** `SynthesisEvalConsumer` writes `SynthesisEvaluated`; `qualityScore` becomes non-null and renders beside the conclusion in the App UI.
- **Done when:** a numeric quality score is visible.

## J5 — Background simulator runs a debate unattended

- **Preconditions:** service running; no UI interaction.
- **Steps:** wait one `DebateSimulator` tick (~30 s).
- **Expected:** a canned topic from `debate-topics.jsonl` is enqueued, a workflow starts, and a debate completes through to `CONCLUDED`.
- **Done when:** a debate the user never submitted appears `CONCLUDED` in the list.
