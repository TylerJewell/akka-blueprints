# User journeys — core-semantic-router

## J1 — HR query: end-to-end handoff to HrSpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `QuerySimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated request seeded as HR-flavoured to drop.

**Expected:**
- The query appears with status `RECEIVED` and within ~1 s transitions to `SANITIZED`. The redacted subject is visible; the raw subject is not.
- Within ~10 s the routing block shows `domain = HR`, `confidence = high`, and a one-sentence reason. The status pill changes to `CLASSIFIED` then `AUTHORIZED`.
- The guardrail block in the right column shows a green "authorized" check with an empty rejections list.
- Within ~5 s of authorization, the right column populates with the `HrSpecialist` answer: a 2–4 paragraph body, an action chip (typically `POLICY_CITED` or `PROCESS_EXPLAINED`), and an `hr` specialist tag.
- The status pill transitions to `ANSWERED`. `finishedAt` is set.
- Within ~10 s of the routing decision, the routing score chip shows a number 1–5 with a rationale.

## J2 — Finance query: end-to-end handoff to FinanceSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a Finance-flavoured request (the seeded JSONL covers the three domains in rotation).

**Expected:**
- Routing emits `domain = FINANCE`. Status transitions through `CLASSIFIED` → `AUTHORIZED` → `ROUTED_FINANCE`.
- `FinanceSpecialist` returns a `QueryAnswer` with a concrete process explanation (likely `PROCESS_EXPLAINED` or `POLICY_CITED`).
- Guardrail shows authorized. Status `ANSWERED`.
- `HrSpecialist` is never invoked for this query — no answer from it appears in any surface.

## J3 — Ambiguous query: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately vague one-liner.

**Steps:**
1. Wait for the ambiguous request to drop.

**Expected:**
- Intent router emits `domain = AMBIGUOUS` with `confidence = low`.
- The workflow calls `RoutingGuardrail`. The guardrail blocks with rejections `["ambiguous-domain", "low-confidence"]`. Status transitions to `BLOCKED`.
- Neither specialist is invoked; the right column shows a muted "Blocked — no specialist invoked" block with the rejection list.
- A `RoutingScored` event still fires; the score chip appears (typically high, since the classifier was right to signal ambiguity).

## J4 — Guardrail blocks a low-confidence routing

**Preconditions:** The seeded JSONL includes one query engineered to produce a low-confidence classification on a recognized domain.

**Steps:**
1. Wait for the low-confidence query to drop and be classified as `HR` or `FINANCE` with `confidence = low`.

**Expected:**
- Status reaches `CLASSIFIED`.
- `RoutingGuardrail` fires with rejection `low-confidence`.
- Status transitions to `BLOCKED`. `finishedAt` is set.
- The right column shows the guardrail block in red with the rejection listed and a muted "no specialist invoked" note.
- The specialist for that domain is never called — the `RouterWorkflow` ends without proceeding to the answer step.

## J5 — Unauthorized-channel routing is blocked

**Preconditions:** A query submitted via the `slack` channel targeting `FINANCE` (which is not an approved channel for Finance queries per the guardrail rubric).

**Steps:**
1. Use `POST /api/queries` with `{ "requesterId": "emp-001", "channel": "slack", "subject": "Invoice processing question", "body": "..." }`.

**Expected:**
- The query is classified as `FINANCE` with medium or high confidence.
- `RoutingGuardrail` fires with rejection `unauthorized-channel`.
- Status transitions to `BLOCKED`.
- No `FinanceSpecialist` is invoked.

## J6 — Routing score appears on every classified query

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new query through the classify step.

**Expected:**
- Within ~10 s of `IntentRouted`, the per-query routing score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail.
- The chip never changes the workflow's flow — a low score does not block the answer from being published.
- For queries that reach `BLOCKED` before routing (domain = AMBIGUOUS or low-confidence), no score chip appears because `IntentRouted` was never emitted.
