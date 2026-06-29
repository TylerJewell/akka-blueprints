# User journeys — sdlc-technical-designer

## J1 — Submit a notification-service feature and get a design proposal

**Preconditions:** Service running on declared port (`http://localhost:9369/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9369/` → App UI tab.
2. From the **Project context** dropdown, pick `event-driven`.
3. Click **Load seeded example** to fill the feature title and the feature description textarea with the notification-service seed.
4. Click **Submit for design**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `CONTEXT_LOADED` within 1 s. The right-pane detail shows the resolved project context: architecture pattern `event-driven`, existing components, preferred patterns.
- Within 30 s the card reaches `PROPOSAL_RECORDED`. The right pane shows: an executive summary paragraph, a component table (at least 2 components including at least one `Consumer` or `EventSourcedEntity`), a data model with at least one entity and at least two fields per entity, an API surface with at least one endpoint, and a decision log with one entry per component.
- Within 1 s of `PROPOSAL_RECORDED`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a malformed proposal

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `design-feature.json` includes deliberately malformed entries (a decision-log entry whose `componentId` is not in the `components` list; a component with `componentKind` outside the allowed set).

**Steps:**
1. Submit any seeded feature description three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/designs/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed proposal.
- The `before-agent-response` guardrail rejects it. The malformed proposal NEVER lands in `DesignRequestEntity` — there is no `ProposalRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed proposal. The card transitions to `PROPOSAL_RECORDED` with a proposal that satisfies all six guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J3 — Thin rationale flags eval score 1

**Preconditions:** Mock LLM mode. A specific mock-response entry has all `DecisionLogEntry.rationale` and `ComponentChoice.rationale` fields set to empty strings.

**Steps:**
1. Submit any seeded feature description and project context whose seed value maps to the "thin-rationale" mock entry.
2. Wait for `EVALUATED`.

**Expected:**
- The proposal lands well-formed (the guardrail checks structural validity, not rationale depth).
- The eval score chip shows **1** and the rationale reads something like "Component and decision-log rationale fields are empty; the proposal is not self-explanatory."
- The card's border highlights red. The engineer knows to request a revised proposal before acting.

## J4 — Context-aware component selection for event-driven profile

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the metrics-aggregator seeded feature description with the `event-driven` project context.
2. Wait for `EVALUATED`.

**Expected:**
- The proposal's `components` list includes at least one `Consumer` and at least one `EventSourcedEntity`.
- The proposal's `preferredPatterns` in the project context (`cqrs`, `event-sourcing`) are reflected: at least one `View` component appears in the component list.
- No component uses a `componentKind` outside the allowed set — confirming the guardrail was not needed and the first iteration was valid.

## J5 — Monolith context produces a structurally different proposal

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the document-processing-pipeline seeded feature description with the `monolith` project context.
2. Wait for `EVALUATED`.
3. Submit the same feature description with the `microservices` context.
4. Wait for `EVALUATED`.

**Expected:**
- The `monolith` proposal has fewer total components (≤ 4) and no `Consumer` components.
- The `microservices` proposal has more components (≥ 5) and includes at least one `HttpEndpoint` and one `View`.
- Both proposals have eval scores ≥ 3, confirming the agent adapted its output to the context rather than returning a generic template.

## J6 — Decision-log completeness check

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit any seeded feature description.
2. Wait for `EVALUATED`.
3. Inspect the proposal's `decisionLog` in the right-pane detail.

**Expected:**
- Every `componentId` that appears in the `components` list also appears as a `componentId` in at least one `decisionLog` entry.
- Every `decisionLog` entry's `alternativesConsidered` list is non-empty.
- The eval score reflects any entries where alternatives were sparse (score 3 or lower if more than half of entries have only one alternative).
