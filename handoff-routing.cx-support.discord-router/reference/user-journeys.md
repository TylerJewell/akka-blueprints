# User journeys — discord-router

## J1 — Community message: end-to-end handoff to CommunitySpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `MessageSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated message seeded as community-flavoured to drop.

**Expected:**
- The message appears with status `RECEIVED` and within ~1 s transitions to `SANITIZED`. The redacted content preview is visible; the raw author handle is not.
- Within ~10 s the routing block shows `category = COMMUNITY`, `confidence = high`, and a one-sentence reason. The status pill changes to `CLASSIFIED` then `ROUTED_COMMUNITY`.
- Within ~5 s of routing, the right column populates with the `CommunitySpecialist` draft: a reply body and an action chip (typically `ANSWERED` or `ACKNOWLEDGED`), and a `community` specialist tag.
- The guardrail block shows a green "allowed" check.
- The status pill transitions to `PUBLISHED`. `finishedAt` is set.
- Within ~10 s of the classification decision, the routing score chip shows a number 1–5 with a rationale.

## J2 — Technical message: end-to-end handoff to TechnicalSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a technical-flavoured message (the seeded JSONL covers all categories in rotation).

**Expected:**
- Routing emits `category = TECHNICAL`. Status `ROUTED_TECHNICAL`.
- `TechnicalSpecialist` returns a `BotReply` with a concrete answer (typically `ANSWERED`, `DOCS_LINKED`, or `FOLLOW_UP_REQUESTED`).
- Guardrail allows. Status `PUBLISHED`.
- The `CommunitySpecialist` is never invoked for this message — no draft from it appears in any surface.

## J3 — Ambiguous message: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous single-token message.

**Steps:**
1. Wait for the ambiguous message to drop.

**Expected:**
- Routing emits `category = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `MessageEscalated`. Status `ESCALATED`.
- Neither specialist is invoked; the right column shows a muted "Escalated — no specialist invoked" block.
- `escalationReason` is populated with the routing reason.
- A `RoutingScored` event still fires; the score chip appears (typically high, since the classifier was correct to refuse).

## J4 — Guardrail blocks a draft with an unapproved external link

**Preconditions:** The seeded JSONL includes a message engineered to produce a draft linking to an external site (and the mock-responses file includes a matching trip-the-guardrail entry).

**Steps:**
1. Wait for the trip message to drop and be classified as `COMMUNITY`.
2. Wait for `CommunitySpecialist` to produce a draft.

**Expected:**
- Status reaches `REPLY_DRAFTED`.
- The guardrail block in the UI shows a red badge with the violation `unapproved-external-link`.
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked draft (not published) and an Unblock button.

## J5 — Moderator unblocks a previously-blocked message

**Preconditions:** A message in `BLOCKED`.

**Steps:**
1. Click the Unblock button on the blocked message.
2. Enter a note ("moderator override — link verified as safe").
3. Confirm.

**Expected:**
- A `POST /api/messages/{id}/unblock` is sent.
- The message transitions to `PUBLISHED`. The previously-blocked draft is now the published `BotReply`.
- The guardrail block continues to show the original violation list (the audit record is preserved); a small note shows the moderator's override.

## J6 — Routing score appears on every classified message

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new message through the classification step.

**Expected:**
- Within ~10 s of `MessageClassified`, the per-message routing score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail.
- The chip never affects the workflow's flow — a low score does not block the reply from being published.
