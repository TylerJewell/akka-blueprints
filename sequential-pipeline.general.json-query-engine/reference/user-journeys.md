# User journeys — json-query-engine

## J1 — Submit a question and get a grounded answer

**Preconditions:** Service running on declared port (`http://localhost:9228/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded document `saas-pricing-catalog` has a matching `src/main/resources/sample-data/documents/saas-pricing-catalog.json` file and schema.

**Steps:**
1. Open `http://localhost:9228/` → App UI tab.
2. From the document dropdown, select `saas-pricing-catalog`. Click the suggested question link "What is the price of the Pro plan?" to fill the question field.
3. Click **Run query**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `PARSING` within 1 s more.
- Within ~20 s the card reaches `PARSED`. The right pane shows the Parsed-question panel with intent type `lookup`, target entity `Pro plan price`, and candidate root keys listing `plans` and `pricing`.
- Within ~20 s more the card reaches `TRAVERSED`. The right pane shows the Traversal panel with ≥ 1 matched path/value row; the path expression begins with `$` and the resolved value is a non-empty string.
- Within ~20 s more the card reaches `RESPONDED`, then `EVALUATED` within 1 s of that. The right pane shows the Answer panel with a complete sentence (≥ 20 chars) and ≥ 1 citation. Every citation's value matches a resolved value in the traversal table. The accuracy score chip shows ≥ 4/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Path-expression guardrail blocks a malformed path

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `traverse-document.json` includes one entry whose `tool_calls` array contains a `resolvePathExpression` call with a path that does not begin with `$` (e.g., `plans.Pro.price`).

**Steps:**
1. Submit the question "What is the price of the Pro plan?" against `saas-pricing-catalog` three times in a row.
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/queries/sse`).

**Expected:**
- On the third submission's `traverseStep`, the agent's first iteration calls `resolvePathExpression` with the malformed path. `PathGuardrail` rejects it with reason `"path-violation: expression must begin with '$', saw 'plans.Pro.price'"`.
- A `GuardrailRejected{phase: "TRAVERSE", tool: "resolvePathExpression", pathExpr: "plans.Pro.price", reason: "..."}` event lands on the entity.
- The malformed path NEVER reaches `TraverseTools.resolvePathExpression` — there is no log line from the tool body for that expression.
- The agent's second iteration corrects the path to `$.plans[?(@.name=='Pro')].price`. The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The path-rejection log strip shows the one rejected call with its full structured reason and the original malformed expression.

## J3 — Ungrounded citation flags accuracy score 1

**Preconditions:** Mock LLM mode. The mock's `compose-response.json` includes one entry whose `Citation.value` does not appear in the paired `TraversalResult.matches[].resolvedValue`.

**Steps:**
1. Submit any seeded question six times. (The ungrounded-citation entry is selected once in every six runs by the mock's `seedFor(queryId)` modulo.)
2. Watch the live list until a query card's border highlights red.

**Expected:**
- The flagged query lands well-formed (the guardrail only checks path validity and phase order, not citation grounding).
- The accuracy score chip shows **1** (or 2 depending on which other rules pass) and the rationale reads: *"Answer grounding failed: citation value 'X' does not appear in the recorded TraversalResult matches."*
- The card border highlights red. The reader knows to inspect this answer before acting on it.
- The other five queries in the run scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded question against any seeded document.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the queryId.

**Expected:**
- The PARSE task's log entries show only `identifyQueryIntent` and `selectRootKeys` calls.
- The TRAVERSE task's log entries show only `resolvePathExpression` and `extractValueAt` calls.
- The RESPOND task's log entries show only `formatAnswer` and `buildCitations` calls.
- No cross-phase calls appear. (If the mock-LLM violation path fires, a single `guardrail.reject` line precedes the agent's retry; the rejected call is logged but never executed.)
- The order of tasks in the log is PARSE → TRAVERSE → RESPOND. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Root-key validation blocks an out-of-schema path

**Preconditions:** Service running with the mock LLM selected. The mock's `traverse-document.json` includes one entry whose `resolvePathExpression` call references a root key absent from the target document's schema (e.g., `$.secrets.apiKey` against `saas-pricing-catalog` whose schema rootKeys are `["plans", "pricing", "features"]`).

**Steps:**
1. Submit any question against `saas-pricing-catalog` four times.
2. Watch for a query that has a path-rejection red dot.

**Expected:**
- The `PathGuardrail` rejects the call with reason `"path-violation: root key 'secrets' not in DocumentSchema.rootKeys for saas-pricing-catalog"`.
- The `GuardrailRejected` event includes the rejected path expression in `pathExpr`.
- The agent retries with a path whose first segment is `plans`, `pricing`, or `features`. The query completes with `EVALUATED` and a score ≥ 3.

## J6 — Question with no matching values in document

**Preconditions:** Service running. Any model provider. A question whose answer cannot be found in the selected document.

**Steps:**
1. In the App UI, type the question "What is the CEO's home address?" and select `saas-pricing-catalog` from the document dropdown.
2. Click **Run query**.

**Expected:**
- `TraverseTools.resolvePathExpression` returns an empty list for all generated path expressions (the document contains no address data).
- The TRAVERSE task returns a `TraversalResult` with `matches = []`.
- The RESPOND task returns a `QueryResult` with `answer = "(no matching values found in document)"` and `citations = []`.
- The accuracy score chip shows 1 (no paths matched, so path coverage fails; all other rules are vacuously satisfied or irrelevant). The rationale names "path coverage failed: no matches for any candidate root key."
- The pipeline completes; nothing crashes; the empty result is honestly empty.
