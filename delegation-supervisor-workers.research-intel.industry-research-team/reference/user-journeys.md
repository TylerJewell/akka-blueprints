# User journeys

Acceptance tests. Each journey lists preconditions, steps, expected state changes, and what passes as "done."

## J1 — Submit a request and watch it synthesize

- **Preconditions:** service running on `http://localhost:9670/`; a model provider resolved (real key or mock).
- **Steps:**
  1. In the App UI tab, type "competitive outlook for grid-scale battery storage" and click Submit (or `POST /api/research-request`).
  2. Watch the brief card via SSE.
- **Expected:** the brief appears in `PLANNED`, moves to `ANALYZING` with 2–4 sector chips, then to `COMPLETED` with a non-empty sanitized body within ~60s.
- **Done:** `GET /api/briefs/{id}` returns `status = COMPLETED`, `sanitizedBody` non-null, `sectors.length` between 2 and 4, `findings.length == sectors.length`.

## J2 — Ungrounded synthesis is blocked

- **Preconditions:** service running; mock provider configured to return a `SynthesizedBrief` with `groundingScore < 0.6` (or a real model that produces a weakly-grounded body).
- **Steps:**
  1. Submit a request whose synthesis does not match the worker findings.
- **Expected:** the brief moves to `BLOCKED` with a non-empty `blockedReason`; it never reaches `COMPLETED`.
- **Done:** `status == BLOCKED`, `blockedReason` present, `sanitizedBody` null.

## J3 — Disclaimer present on every completed brief

- **Preconditions:** at least one brief has reached `COMPLETED`.
- **Steps:**
  1. Read the completed brief's `sanitizedBody`.
- **Expected:** the body ends with the configured sector disclaimer text.
- **Done:** `sanitizedBody` contains the disclaimer string from `eval-matrix.yaml` control S1.

## J4 — Background load from the simulator

- **Preconditions:** service running; no UI interaction.
- **Steps:**
  1. Wait one `RequestSimulator` tick (~30s).
- **Expected:** a brief seeded from `sample-events/research-requests.jsonl` runs from request to `COMPLETED` (or `BLOCKED`) with no human action.
- **Done:** `GET /api/briefs` shows at least one terminal brief that no client submitted.

## J5 — Eval Matrix and Risk Survey render

- **Preconditions:** service running.
- **Steps:**
  1. Open the Eval Matrix tab, then the Risk Survey tab.
- **Expected:** Eval Matrix shows 2 controls (G1 guardrail, S1 sanitizer) as `matrix-card`/`matrix-row` with colored mechanism pills; Risk Survey shows pre-filled answers with `TO_BE_COMPLETED_BY_DEPLOYER` fields muted.
- **Done:** both tabs render in the matrix style, no JSON dump, no blank panel.
