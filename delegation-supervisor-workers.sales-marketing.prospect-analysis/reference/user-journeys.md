# User journeys

Acceptance journeys. Each must pass for the blueprint to be "generated correctly."

## J1 — Analyze a company end-to-end
**Preconditions:** service running; a model provider (real or mock) configured.
**Steps:**
1. In the App UI, enter a company name and click Analyze.
2. Observe the prospect appear via SSE.
**Expected:** the prospect transitions `RESEARCHING` → `ANALYZING` → `STRATEGIZING` → `COMPLETED`. Each phase populates its result: a `companyProfile`, an `orgStructureSummary` with a non-empty `decisionMakers` list, and an `outreachStrategy`. The whole run completes within a few minutes.

## J2 — Off-domain research blocked
**Preconditions:** allow-list configured in the guardrail; a request whose research would target an off-list domain.
**Steps:**
1. Submit a company whose `domain` is outside the allow-list (or force an off-list tool call).
2. Watch the research phase.
**Expected:** the before-tool-call guardrail refuses the off-domain call; the call never reaches `ResearchToolEndpoint`. The research worker proceeds without that source (noting the gap), and the refusal is visible. No off-list fetch occurs.

## J3 — Contact details masked
**Preconditions:** a prospect that reaches `ANALYZING`.
**Steps:**
1. Expand a prospect that has completed analysis.
2. Inspect the decision-maker list in the UI and `GET /api/prospects/{id}`.
**Expected:** every `contactHint` is masked. No raw contact value appears in the read model, the SSE stream, or the UI.

## J4 — Stalled analysis
**Preconditions:** a prospect that enters an in-progress state and does not advance (for example, a worker that does not complete).
**Steps:**
1. Leave a prospect in `RESEARCHING`, `ANALYZING`, or `STRATEGIZING` past the 3-minute threshold.
**Expected:** `StalledAnalysisMonitor` marks it `STALLED`; the UI status chip updates accordingly.

## J5 — Background load from the simulator
**Preconditions:** service running; no UI interaction.
**Steps:**
1. Start the service and wait.
**Expected:** `RequestSimulator` seeds canned companies from `sample-events/analysis-requests.jsonl` every 30s; each one starts a fresh workflow and appears in the list.
