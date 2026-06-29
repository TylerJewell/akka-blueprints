# User journeys — aws-ops-assistant

## J1 — Submit a read-only EC2 inventory request

**Preconditions:** Service running on `http://localhost:9276/`; a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9276/` → App UI tab.
2. From the **Request scope** dropdown, pick `Read Only`.
3. Click **Load seeded example** and select "EC2 inventory" to fill the request textarea.
4. Click **Submit request**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `PLANNING` within 1 s. No confirmation cards appear — all planned calls are READ.
- The card transitions to `EXECUTING`, then `COMPLETED` within 30 s.
- The right-pane action results table shows at least one COMPLETED row for `EC2.DescribeInstances` with a non-empty `responseSnippet`.
- The report section shows status badge `COMPLETED`, a 1–3-sentence summary, and `completedAt`.
- No confirmation cards were shown at any point (read-only scope, no mutating calls).

## J2 — Submit a mutating EC2 resize request with HITL confirmation

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Request scope** dropdown, pick `Mutating`.
2. Click **Load seeded example** and select "EC2 resize" to fill the request.
3. Click **Submit request**.
4. When the confirmation card appears for `EC2.ModifyInstanceAttribute`, read the resource ARN and rationale.
5. Click **Approve**.

**Expected:**
- The card transitions through `SUBMITTED → PLANNING → AWAITING_CONFIRMATION` within 10 s.
- A confirmation card is visible in the right pane with: `EC2` service badge, `ModifyInstanceAttribute` API call, a resource ARN containing `i-0abc123` (the seeded instance id), `MUTATING` kind chip, and a one-sentence rationale.
- After clicking **Approve**, the card transitions to `EXECUTING` within 2 s.
- The card transitions to `COMPLETED` within 30 s. The action results table shows: one COMPLETED `DescribeInstances` row (READ, no confirmation) followed by one COMPLETED `ModifyInstanceAttribute` row (MUTATING, APPROVED).
- The `decisions` list in `GET /api/ops-requests/{id}` contains one entry with `outcome: APPROVED` and the `decidedBy` value from the `submittedBy` field.

## J3 — Operator halts an in-flight multi-action request

**Preconditions:** Service running. Any model provider. The seeded "S3 storage report" request is used (it generates 3 or more sequential DescribeInstances/ListBuckets/GetBucketLocation calls to give the operator time to halt).

**Steps:**
1. Submit the "S3 storage report" seeded request (READ_ONLY scope, 3+ planned calls).
2. While the card is in `EXECUTING` state (visible from the status pill), click the red **Halt** button in the right pane (or POST `{ "haltedBy": "operator" }` to `/api/ops-requests/{id}/halt`).

**Expected:**
- The card transitions to `HALTED` within 2 s of the halt POST.
- The right pane shows the red "This request was halted. Partial results below." banner.
- The report section shows status badge `HALTED` and the partial `actions` list — only the actions that had already completed before the halt flag was observed appear in the list.
- No further AWS API calls are logged after the halt timestamp.
- `GET /api/ops-requests/{id}` returns `status: "HALTED"` and `finishedAt` set to the halt time.

## J4 — Guardrail blocks a disallowed AWS service target

**Preconditions:** Service running with the default allow-list (`["EC2", "S3", "Lambda", "IAM", "CloudWatch"]`). Mock LLM mode or a real model.

**Steps:**
1. Submit a custom request: "List all DynamoDB tables and describe the largest one" with scope `Mixed`.
2. Wait for the report.

**Expected:**
- The card reaches `COMPLETED` (not FAILED — the agent handled the block gracefully).
- The report status badge is `BLOCKED`.
- The action results table contains at least one row with outcome `BLOCKED`, `awsService = "DynamoDB"`, and a `responseSnippet` starting with `"blocked-resource: DynamoDB"`.
- No `DynamoDB.*` API call appears in the service log — the guardrail prevented it before any network traffic.
- The summary paragraph mentions that DynamoDB access was blocked.

## J5 — Operator declines a mutating action; request completes with SKIPPED entry

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the "EC2 resize" seeded request (scope: Mutating).
2. When the confirmation card appears for `EC2.ModifyInstanceAttribute`, click **Decline**.

**Expected:**
- The card transitions from `AWAITING_CONFIRMATION` back to `EXECUTING` within 2 s of the Decline click.
- The card reaches `COMPLETED` within 30 s (the resize was declined, but the preceding read call completed, so the overall request is not FAILED).
- The action results table shows: one COMPLETED `DescribeInstances` row and one SKIPPED `ModifyInstanceAttribute` row.
- The `decisions` list in `GET /api/ops-requests/{id}` contains one entry with `outcome: DECLINED`.
- The summary paragraph states that the resize was skipped at operator request.

## J6 — Confirmation window expires; action is auto-skipped

**Preconditions:** Service running. Mock LLM mode. The `confirmStep` timeout is set to a short value (override `akka.javasdk.dev-mode.confirmation-timeout-seconds = 15` in application.conf for testing, or wait 10 minutes in default config).

**Steps:**
1. Submit the "EC2 resize" seeded request.
2. When the confirmation card appears, do nothing — do not click Approve or Decline.
3. Wait for the `confirmStep` timeout to expire.

**Expected:**
- After the timeout, the card transitions from `AWAITING_CONFIRMATION` to `EXECUTING` automatically.
- The timed-out action appears in the action results table with outcome `SKIPPED` and a `responseSnippet` of `"Confirmation window expired; action skipped."`.
- The card reaches `COMPLETED` with report status `COMPLETED` (a skipped action is not a failure).
