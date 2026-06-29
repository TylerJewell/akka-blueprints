# User journeys — merchandiser

Six numbered acceptance journeys. Each defines preconditions, steps, and expected outcomes. `/akka:implement` and manual QA both use these as the acceptance bar.

---

## J1 — Happy path: submit, generate, approve, publish

**Preconditions:** Service is running. No prior proposals in the system. Seasonal-promotion seed objective is available.

**Steps:**
1. Open the App UI tab.
2. Click "Load seeded example" for the seasonal-promotion objective (outdoor leisure catalog, all products in scope).
3. Click **Submit objective**.
4. Observe the proposal card appear in the live list with status `SUBMITTED`.
5. Within 2 s the card transitions to `CONTEXT_LOADED`. The right pane shows the catalog context: product count, category names, active promotion count.
6. Within 30 s the card transitions through `GENERATING` to `PENDING_APPROVAL`. The right pane shows the proposal summary and change-recommendation table (at least one `ChangeRecommendation` per in-scope product).
7. Enter `merchandiser-qa` in the **Decided by** field and click **Approve**.
8. The card transitions through `PUBLISHING` to `PUBLISHED`. The approval decision block appears showing approver, timestamp, and (blank) note.

**Expected:**
- `GET /api/proposals/{id}` returns `status: "PUBLISHED"`, `approval.status: "APPROVED"`, `approval.decidedBy: "merchandiser-qa"`.
- Every `ChangeRecommendation` in `proposal.changes` has a non-empty `targetRef`, `currentValue`, `proposedValue`, and `rationale`.

---

## J2 — Guardrail blocks write tool call during generation

**Preconditions:** Mock LLM path active (option a). The mock is seeded to attempt a `setPromotion` tool call on the third proposal's first iteration (modulo seed).

**Steps:**
1. Submit two proposals with any objective to advance the seed counter past the first two.
2. Submit a third proposal (inventory-clearance seed).
3. Observe the `GENERATING` status on the card.
4. Check the server log: confirm `StorefrontGuardrail` emitted a `blocked-write-tool` rejection for `setPromotion`.
5. The card eventually transitions to `PENDING_APPROVAL` with a `MerchandisingProposal` that includes a `PROMOTION_CREATE` `ChangeRecommendation` for the intended promotion — not a live tool call.

**Expected:**
- No `setPromotion` write executed during generation.
- The final proposal's `changes` list contains a `ChangeRecommendation` of type `PROMOTION_CREATE` covering the intended promotion, with a rationale explaining what was intended.
- `proposal.status` reaches `PENDING_APPROVAL`, not `FAILED`.

---

## J3 — Merchant rejects a proposal

**Preconditions:** A proposal in `PENDING_APPROVAL` state exists (from J1 or a fresh submission).

**Steps:**
1. Select the pending proposal in the live list.
2. Enter `merchandiser-bob` in the **Decided by** field, enter `"Description changes are too vague — needs product-specific benefit claims."` in the **Note** field.
3. Click **Reject**.
4. The card transitions to `REJECTED`. The right pane shows the rejection decision block with the note.

**Expected:**
- `GET /api/proposals/{id}` returns `status: "REJECTED"`, `approval.status: "REJECTED"`, `approval.decidedBy: "merchandiser-bob"`, `approval.note` contains the entered text.
- The original `proposal.changes` array is still present on the entity — rejection does not erase the agent output.
- The approve and reject controls are no longer visible on the card.

---

## J4 — Approval timeout expires

**Preconditions:** Service is running with `akka.javasdk.agent.approval-timeout-seconds` set to a short test value (e.g., 60 s via an env override or a test configuration).

**Steps:**
1. Submit a proposal and wait for it to reach `PENDING_APPROVAL`.
2. Do not approve or reject. Wait for the timeout to fire.
3. Observe the card transition to `EXPIRED` in the live list.

**Expected:**
- `GET /api/proposals/{id}` returns `status: "EXPIRED"`, `finishedAt` is set, `approval` is `null`.
- The UI shows an amber expiry notice on the card instead of approval controls.

---

## J5 — SKU-list scope produces targeted recommendations

**Preconditions:** Service is running. Skincare clearance seed objective with a 3-SKU list available.

**Steps:**
1. Load the inventory-clearance seeded example (SKU_LIST scope, 3 SKUs from the skincare catalog).
2. Click **Submit objective**.
3. Wait for `PENDING_APPROVAL`.

**Expected:**
- `proposal.changes` contains exactly 3 `ChangeRecommendation` entries (one per SKU in scope).
- No recommendation targets a SKU outside the submitted `skuList`.
- Each recommendation's `targetRef` matches one of the three submitted SKUs exactly.

---

## J6 — SSE stream delivers real-time state transitions

**Preconditions:** Service is running. A browser or `curl` client is subscribed to `GET /api/proposals/sse`.

**Steps:**
1. Open an SSE connection to `GET /api/proposals/sse` in a separate browser tab or via `curl -N`.
2. Submit a new proposal.
3. Observe the SSE stream in the connected client.

**Expected:**
- The client receives at minimum these events in order: `SUBMITTED`, `CONTEXT_LOADED`, `GENERATING`, `PENDING_APPROVAL`.
- Each event carries `proposalId`, `status`, and the full row state at the moment of transition (per the SSE format in `api-contract.md`).
- A late-joining client that subscribes after `CONTEXT_LOADED` still receives `GENERATING` and `PENDING_APPROVAL` as they occur — it does not need to replay history to stay current.
