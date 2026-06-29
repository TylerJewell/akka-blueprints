# User journeys — it-helpdesk

## J1 — Access request: end-to-end handoff to AccessSpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `RequestSimulator` enabled.

**Steps:**
1. Open `http://localhost:9335/` → App UI tab.
2. Wait up to 30 s for the first simulated request seeded as access-flavoured to drop.

**Expected:**
- The request appears with status `RECEIVED` and within ~1 s transitions to `SANITIZED`. The redacted subject is visible; the raw subject is not. If the seeded request contained a password, the body shows `[REDACTED-SECRET]` and the secret category chip lists `"password"`.
- Within ~10 s the classification block shows `category = ACCESS`, `confidence = high`, and a one-sentence reason. Status transitions to `CLASSIFIED` then `ROUTED_ACCESS`.
- The right column populates with the `AccessSpecialist` resolution: a `Re: …` subject, a 2–4 paragraph body, an action chip (`TICKET_FILED` or `SELF_SERVICE_RESOLVED`), and a specialist tag of `"access"`.
- If a `ProposedTicket` is present: the filed-ticket card shows the title, `access-team` assignee, and priority badge. The guardrail block shows a green "allowed" check.
- Status transitions to `RESOLVED`. `finishedAt` is set.
- Within ~10 s of the classification decision, the routing score chip shows a number 1–5 with rationale.

## J2 — Infrastructure request: end-to-end handoff to InfraSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop an infrastructure-flavoured request.

**Expected:**
- Classification emits `category = INFRASTRUCTURE`. Status `ROUTED_INFRASTRUCTURE`.
- `InfraSpecialist` returns a `Resolution` with action `TICKET_FILED` or `RUNBOOK_LINKED`.
- Guardrail allows (or is skipped if no `ProposedTicket`). Status `RESOLVED`.
- `AccessSpecialist` and `SoftwareSpecialist` are never invoked for this request.

## J3 — Software request: end-to-end handoff to SoftwareSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Wait for the simulator to drop a software-flavoured request.

**Expected:**
- Classification emits `category = SOFTWARE`. Status `ROUTED_SOFTWARE`.
- `SoftwareSpecialist` returns a `Resolution` with action `SELF_SERVICE_RESOLVED` or `RUNBOOK_LINKED`.
- Status `RESOLVED`. Routing score chip appears within ~10 s.

## J4 — Ambiguous request: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the ambiguous request to drop.

**Expected:**
- Classification emits `category = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `RequestEscalated`. Status `ESCALATED`.
- No specialist is invoked; the right column shows a muted "Escalated — no specialist invoked" block with the `escalationReason`.
- A `RoutingScored` event still fires; the score chip appears (typically high, since the classifier correctly refused to route).

## J5 — Credential in request body is redacted before LLM

**Preconditions:** The seeded JSONL includes a request whose body contains a password token.

**Steps:**
1. Wait for the credential-containing request to drop.

**Expected:**
- The sanitized body displayed in the centre column shows `[REDACTED-SECRET]` where the credential was.
- The secret category chip in the list row lists `"password"` (or the appropriate detected category).
- The centre column's redacted-body block does not show any raw credential text.
- The workflow proceeds normally — sanitization does not prevent classification or resolution.

## J6 — Guardrail blocks an open-ended admin ticket proposal

**Preconditions:** The seeded JSONL includes a request engineered to coax an open-ended admin ticket out of a specialist (and the mock-responses file includes a matching guardrail-tripping entry).

**Steps:**
1. Wait for the trip request to drop and be classified as `ACCESS` or `INFRASTRUCTURE`.
2. Wait for the specialist to produce a `Resolution` with a `ProposedTicket` requesting blanket admin access.

**Expected:**
- Status reaches `RESOLUTION_DRAFTED`.
- The guardrail block in the UI shows a red badge with the violation `open-ended-admin-access`.
- Status transitions to `TICKET_BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked proposed ticket (not filed) and an Unblock button.
- The technician clicks Unblock, enters a note ("approved by CTO"), and confirms.
- A `POST /api/requests/{id}/unblock` is sent. The request transitions to `RESOLVED`. The audit record preserves the original violation list alongside the override note.
