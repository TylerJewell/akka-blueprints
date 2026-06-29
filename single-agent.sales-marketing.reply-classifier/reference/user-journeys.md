# User journeys — reply-classifier

Acceptance bar for the generated system. Each journey defines a precondition, numbered steps, and the expected outcome.

---

## J1 — Happy path: INTERESTED reply advances deal stage

**Precondition:** The service is running. Deal `deal-00421` exists in stage `PROSPECTING` in the simulated Pipedrive client.

**Steps:**

1. Open the App UI tab.
2. Select the seeded "Interested" reply ("Happy to set up a 30-minute call — does next Thursday work?").
3. Set Deal ID to `deal-00421` and click **Classify reply**.
4. Observe the card enter `CLASSIFYING` state.
5. Wait up to 30 s.

**Expected:**

- The card transitions to `CLASSIFIED` with intent badge `INTERESTED` and a confidence score ≥ 80.
- The card then transitions to `CRM_UPDATED`. The CRM action panel shows `PROSPECTING → QUALIFIED`.
- The entity's `crmResult.success` is `true` and `crmResult.newStage` is `QUALIFIED`.
- No guardrail rejection banner is shown.

---

## J2 — Guardrail fires on an invalid stage transition

**Precondition:** The service is running. Mock LLM is configured (option a). Deal `deal-99001` exists in stage `CLOSED_WON` in the simulated Pipedrive client. The mock reply sequence for this deal id is set so the first iteration proposes `UPDATE_STAGE` with `newStage = QUALIFIED` (a forward mutation on a closed deal).

**Steps:**

1. Submit a reply against deal `deal-99001`.
2. Wait for classification to complete.

**Expected:**

- On the first iteration, `CrmMutationGuardrail` rejects the tool call with reason "Stage transition CLOSED_WON → QUALIFIED is not permitted."
- A guardrail rejection banner appears in the selected-reply detail pane showing the rejection reason and "Iteration 1 of 3."
- On the second iteration, the agent emits `SKIP_CRM_UPDATE` with skip reason referencing the rejection.
- The card transitions to `CRM_SKIPPED`. The guardrail event banner remains visible.
- The entity's `status` is `CRM_SKIPPED`, not `FAILED`.

---

## J3 — UNSUBSCRIBE reply produces CRM_SKIPPED without guardrail firing

**Precondition:** The service is running. Deal `deal-00505` exists in stage `PROSPECTING`.

**Steps:**

1. Select the seeded "Unsubscribe" reply ("Please remove me from your list").
2. Set Deal ID to `deal-00505` and click **Classify reply**.

**Expected:**

- The card transitions to `CLASSIFIED` with intent badge `UNSUBSCRIBE` and confidence score ≥ 95.
- The agent's `crmAction.type` is `SKIP_CRM_UPDATE` with `skipReason` containing "Unsubscribe requested."
- No `updateDealStage` tool call is proposed, so the guardrail never fires.
- The card transitions to `CRM_SKIPPED`.
- No guardrail rejection banner is shown.

---

## J4 — OBJECTION reply is classified and CRM update is skipped with rationale

**Precondition:** The service is running. Deal `deal-00312` exists in stage `QUALIFIED`.

**Steps:**

1. Submit the seeded "Price Objection" reply ("Your pricing is 3× what we're paying now — is there room to negotiate?").
2. Set Deal ID to `deal-00312` and click **Classify reply**.

**Expected:**

- The card transitions to `CLASSIFIED` with intent badge `OBJECTION`.
- The rationale names the specific price signal in the reply.
- The `crmAction.type` is `SKIP_CRM_UPDATE` with `skipReason` referencing the objection type (e.g., "Price objection; awaiting rep follow-up").
- The card transitions to `CRM_SKIPPED`. The deal stage in the simulated client remains `QUALIFIED`.

---

## J5 — Raw reply text preserved for audit; UI shows only outcome

**Precondition:** The service is running.

**Steps:**

1. Submit a reply that contains personal contact details: "Call me at +1-555-867-5309, or email me at bob.walker@example.com. Ready to move forward."
2. Wait for classification to complete.

**Expected:**

- The UI's live list card shows only the intent badge, confidence chip, and deal id — never the raw reply text.
- The selected-reply detail pane's "Raw reply text" block renders the full reply including the phone number and email. This is the auditable record for the sales rep.
- The API's `GET /api/replies/{id}` response includes `submission.rawReplyText` with the full original text.
- The classification's `rationale` may reference the forward-moving signal ("expresses readiness to move forward") without echoing the contact details.

---

## J6 — OUT_OF_OFFICE auto-reply is recognised and skipped

**Precondition:** The service is running. Deal `deal-00789` exists in stage `PROPOSAL`.

**Steps:**

1. Submit the seeded "Out of Office" reply ("I'm out of the office until July 7th. For urgent matters, contact my colleague at backup@example.com.").
2. Set Deal ID to `deal-00789` and click **Classify reply**.

**Expected:**

- The card transitions to `CLASSIFIED` with intent badge `OUT_OF_OFFICE` and confidence score ≥ 90.
- The `crmAction.type` is `SKIP_CRM_UPDATE` with `skipReason` "Auto-reply; no action taken."
- The card transitions to `CRM_SKIPPED`. The deal stage remains `PROPOSAL`.
