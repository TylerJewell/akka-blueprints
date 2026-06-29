# User journeys — durable-agent-baseline

## J1 — Submit a Data Pipeline work order and get a result

**Preconditions:** Service running on declared port (`http://localhost:9629/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9629/` → App UI tab.
2. From the **Work-order template** dropdown, pick `Data Pipeline`.
3. Click **Load example** to fill the title and steps list.
4. Click **Submit**.

**Expected:**
- The new card appears in the live list with status `INITIATED` within 1 s.
- The card transitions to `RUNNING` within 1 s as the workflow's `initStep` completes.
- Within 60 s the card reaches `COMPLETED`. The right pane shows the `WorkOrderResult` — a resolution badge (COMPLETED / PARTIAL / FAILED), the summary paragraph, and one step-outcome row for each of the 4 submitted steps. Every row has a non-empty `output` string and an elapsed time.
- Within 1 s of `COMPLETED`, the card reaches `EVALUATED` and shows a performance score chip (1–5) plus a one-line rationale.

## J2 — Restart resumes from the last completed step

**Preconditions:** Service running with the mock LLM. A work order is in status `RUNNING` (submit one and wait ~2 s for `initStep` to complete and `agentStep` to start).

**Steps:**
1. Submit a 5-step Code Analysis work order.
2. Wait until the right pane shows at least 2 steps as COMPLETED (via SSE updates).
3. Stop the service (Ctrl+C or via the Akka MCP stop command).
4. Restart the service (`/akka:build`).
5. Reconnect the browser to `http://localhost:9629/`.

**Expected:**
- The work order that was in-flight reappears in the live list with status `RUNNING`.
- The steps that had already completed remain COMPLETED in the right pane. The workflow resumes from the first step that had not yet finished — it does not re-execute completed steps.
- The work order proceeds to `EVALUATED` without any manual intervention.
- The entity event log (visible via `GET /api/work-orders/{id}`) shows no duplicate step-start events for the already-completed steps.

## J3 — Runtime monitor detects a stall

**Preconditions:** Service running with the mock LLM configured to delay the `fetch-source-data` step for more than 90 s (achieved by setting a deliberate delay in the mock response for that step id, or by using a model provider that takes longer than the threshold under load).

**Steps:**
1. Submit any 3-step work order that includes `fetch-source-data` as a step.
2. Wait 95 s after the card enters `RUNNING`.
3. Observe the right pane's alert section.

**Expected:**
- An `AlertRecorded` event with `kind = STALL` and `stepId = fetch-source-data` appears in the entity log.
- The alert badge appears on the work-order card in the live list (orange STALL badge).
- The right pane's alert section shows the stall detail with a timestamp.
- The work order is still `RUNNING` — the stall alert is non-blocking. The agent eventually completes or fails on its own.

## J4 — High-retry run receives a low eval score

**Preconditions:** Mock LLM mode. The mock's `process-work-order.json` includes entries where every step has `retryCount >= 2` (the "high-retry" entries).

**Steps:**
1. Submit a work order whose `workOrderId` modulo seed selects a high-retry mock entry (every 4th submission in default seed configuration).
2. Wait for `EVALUATED`.

**Expected:**
- The `WorkOrderResult.outcomes` list shows `retryCount >= 2` on most or all steps.
- The eval score chip shows **2** or lower with a rationale that references high retry density.
- The card's border highlights red. The operator can identify this run as a candidate for investigation.

## J5 — Partial completion is correctly classified

**Preconditions:** Mock LLM mode. A mock entry returns `resolution = PARTIAL` with at least one step `FAILED`.

**Steps:**
1. Submit a 5-step Report Assembly work order that uses the PARTIAL mock entry.
2. Wait for `EVALUATED`.

**Expected:**
- The `WorkOrderResult.resolution` is `PARTIAL`.
- The resolution badge on the card shows `PARTIAL` in yellow.
- The step-outcome rows in the right pane show at least one red FAILED row with a non-empty `output` explaining the failure.
- The eval score is ≤ 3, reflecting that not all steps completed.

## J6 — Aggregate performance view on Eval Matrix tab

**Preconditions:** At least 5 work orders have been submitted and reached `EVALUATED` with varying scores.

**Steps:**
1. Open the Eval Matrix tab.
2. Expand the E1 row (periodic performance evaluator).

**Expected:**
- The expanded row shows the `implementation` paragraph from `eval-matrix.yaml` referencing `PerformanceEvaluator`.
- The App UI tab's live list shows score chips (1–5) on every EVALUATED card.
- Cards with score ≤ 2 have a red border visible in the list without opening the detail pane.
