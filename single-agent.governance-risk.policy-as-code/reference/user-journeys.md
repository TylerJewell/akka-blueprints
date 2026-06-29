# User journeys — policy-as-code

## J1 — Submit an infrastructure change and get a DENY decision

**Preconditions:** Service running on declared port (`http://localhost:9701/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9701/` → App UI tab.
2. From the **Policy set** dropdown, pick `Infrastructure (5 rules)`.
3. Click **Load seeded example** to fill the change title, type, and payload textarea with the Terraform plan seed that contains a public S3 bucket.
4. Click **Submit for evaluation**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `VALIDATED` within 1 s. The right-pane detail shows the normalized payload and affected-resource chips (`aws_s3_bucket.audit-logs`, `aws_s3_bucket_public_access_block.audit-logs`).
- Within 30 s the card reaches `DECISION_RECORDED`. The right pane shows: an outcome badge of `DENY`, the rationale paragraph, and a violation row for the `infra-s3-public-access-block` rule at `CRITICAL` severity with evidence and recommendation text.
- Within 1 s of `DECISION_RECORDED`, the card reaches `GATED` and shows a `BLOCKED` gate chip with reason text. The card border highlights red.

## J2 — Before-tool-call guardrail blocks a disallowed lookup

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `enforce-policy.json` includes entries that instruct the agent to attempt a disallowed tool call on the first iteration.

**Steps:**
1. Submit any seeded change three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the browser dev tools network panel (`/api/changes/sse`).

**Expected:**
- The third submission's first agent iteration attempts to call a disallowed tool (e.g., `fetch-external-policy-store`).
- The `before-tool-call` guardrail blocks it. The blocked tool is never invoked — no network request leaves the service for that tool.
- The agent receives a structured rejection and continues reasoning using the attached change payload and the permitted tools only.
- The card still reaches `DECISION_RECORDED` within the step timeout. The decision is based solely on the attached payload.
- The service log shows one `guardrail.reject` line with the disallowed tool name and the blocking reason.

## J3 — DENY decision causes gate endpoint to return BLOCKED

**Preconditions:** Service running. Any model provider. A DENY decision has been produced (J1 or equivalent).

**Steps:**
1. Note the `changeId` of a completed DENY evaluation from the live list.
2. From a terminal or curl client, issue: `GET http://localhost:9701/api/changes/{changeId}/gate`.

**Expected:**
- Response body: `{ "changeId": "...", "gateStatus": "BLOCKED", "reason": "PolicyDecision.outcome is DENY; deployment is blocked." }`.
- HTTP status 200.
- A CI pipeline step that polls this endpoint and sees `BLOCKED` must fail the pipeline step — the gate is permanent for this change.

## J4 — ALLOW decision produces OPEN gate

**Preconditions:** Service running with the mock LLM. A mock entry that returns an ALLOW decision (no violations) is selected.

**Steps:**
1. From the **Policy set** dropdown, pick `Supply-chain (4 rules)`.
2. Click **Load seeded example** to fill the dependency-version-bump payload (which satisfies all four supply-chain rules).
3. Click **Submit for evaluation**.
4. Wait for `GATED`.

**Expected:**
- The outcome badge shows `ALLOW`. The violations table is empty.
- The CI gate chip shows `OPEN` (green). The card border is not highlighted red.
- `GET /api/changes/{changeId}/gate` returns `{ "gateStatus": "OPEN", "reason": "No DENY outcome and no CRITICAL violations." }`.

## J5 — WARN decision with MEDIUM violations keeps gate OPEN

**Preconditions:** Service running. Any model provider. A change with only MEDIUM-severity violations is submitted.

**Steps:**
1. Submit the Kubernetes seed change with the `Kubernetes (6 rules)` policy set.
2. Wait for `GATED`.

**Expected:**
- The outcome badge shows `WARN`. The violations table has one or more entries with severity `MEDIUM` or lower.
- The CI gate chip shows `OPEN` — a WARN decision does not block the gate.
- The card border is not highlighted red, but the WARN badge signals the operator to inspect before proceeding.

## J6 — Failed evaluation preserves partial state

**Preconditions:** Service running. Mock LLM timeout path or network failure simulated.

**Steps:**
1. With the mock configured to time out on the `enforceStep` for a specific seed (force by stopping the mock LLM process after the workflow starts), submit any change.
2. Wait for the `enforceStep` to exhaust its retries (up to 60 s × 3 = 180 s max; in mock mode the timeout triggers faster).

**Expected:**
- The card transitions through `SUBMITTED` → `VALIDATED` → `ENFORCING` → `FAILED`.
- The right-pane detail shows the validated change preview (the normalized payload), confirming the entity preserved pre-failure data.
- No `DECISION_RECORDED` or `GATED` event appears — the pipeline is effectively blocked by absence of a gate result, which is the safe default.
- The service log shows the `EvaluationFailed` reason with the timeout context.
