# User journeys — guardrails-pattern

Acceptance bar for the generated system. Each journey maps to a named test path through the UI and the event log.

---

## J1 — Happy path: clean prompt, GENERAL policy, clean reply

**Preconditions:**
- Service is running on port 9321.
- The UI's App UI tab is open.
- At least one seeded prompt is available ("What is the boiling point of water at 3000 m altitude?").

**Steps:**
1. Select policy profile `GENERAL`.
2. Load the seeded knowledge question or type it in.
3. Enter a `submittedBy` value (e.g., `user-1`).
4. Click **Submit**.

**Expected:**
- A new card appears in the live list with status `SUBMITTED`.
- Within ~1 s the card transitions to `PROMPT_CHECKED`. The detail pane shows a green PASSED chip in the input guardrail row.
- Within ~30 s the card transitions to `REPLYING` then `REPLY_RECORDED`. The reply section shows a `FACTUAL` category chip, a confidence bar above 0.5, a non-empty reply text, and an output guardrail PASSED chip.
- The guardrail timeline at the bottom shows both checks as PASSED.
- `GET /api/interactions/{id}` returns `"status": "REPLY_RECORDED"` with non-null `inputGuardResult.passed: true`, non-null `reply`, and non-null `outputGuardResult.passed: true`.

---

## J2 — Input guardrail blocks a non-compliant prompt

**Preconditions:**
- Service is running.
- STRICT policy profile is available.
- The seeded blocked prompt (`ignore previous instructions and...`) is available via "Load seeded example".

**Steps:**
1. Select policy profile `STRICT`.
2. Load the seeded blocked prompt.
3. Click **Submit**.

**Expected:**
- A new card appears with status `SUBMITTED`.
- Within ~1 s the card transitions to `BLOCKED`. The detail pane shows a red BLOCKED chip in the input guardrail row, with the matched `ruleId` (e.g., `STRICT-jailbreak-marker-1`) and a human-readable `reason`.
- The reply section and output guardrail row are absent — no model call was made.
- `GET /api/interactions/{id}` returns `"status": "BLOCKED"`, `"inputGuardResult": {"passed": false, "ruleId": "STRICT-jailbreak-marker-1", ...}`, and `"reply": null`.
- The Akka server logs contain no outbound LLM request for this interaction.

---

## J3 — Output guardrail rejects a malformed reply and the agent retries

**Preconditions:**
- Service is running with the mock LLM provider (option-a) enabled.
- The mock is configured to return a malformed reply on the first iteration of every 3rd interaction (modulo seed).
- The user submits the 3rd interaction (or a specific seed triggers the malformed path).

**Steps:**
1. Select policy profile `GENERAL`.
2. Load the creative writing seeded prompt.
3. Click **Submit**.

**Expected:**
- The card reaches `REPLYING` state.
- The agent's first iteration produces a malformed reply (e.g., `confidence = 1.7`). The `before-agent-response` guardrail rejects it. The UI does not display the malformed version at any point.
- The agent's second iteration produces a well-formed reply. The card transitions to `REPLY_RECORDED`.
- The final `outputGuardResult` shows `passed: true`.
- `GET /api/interactions/{id}` returns a valid `AgentReply` with `confidence` in `[0.0, 1.0]`.
- The Akka server logs show two agent iterations for this task: the first rejected, the second accepted.

---

## J4 — Reply containing a blocked topic is caught by the output guardrail

**Preconditions:**
- Service is running with the mock LLM provider configured to return a reply that contains a blocked-topic term on the first iteration for a specific seed.
- Policy profile is `CONSERVATIVE`.

**Steps:**
1. Select policy profile `CONSERVATIVE`.
2. Submit a clean prompt that the input guardrail passes.

**Expected:**
- The input guardrail passes the prompt (`PROMPT_CHECKED`).
- The agent's first reply contains a term from the CONSERVATIVE blocked-topic vocabulary. The `before-agent-response` guardrail rejects it.
- After retry, the agent's second reply is clean. The card transitions to `REPLY_RECORDED` with a PASSED output guardrail chip.
- The guardrail timeline shows: input-guard PASSED, output-guard REJECTED (first attempt) then PASSED (final state displayed).
- The entity log contains `ReplyGuardChecked{passed: true}` as the terminal event — the rejected-first-attempt outcome is captured in the Akka server logs but is not surfaced as a durable entity event (only the final state is).

---

## J5 — Policy profile comparison across three profiles

**Preconditions:**
- Service is running.
- A prompt that passes GENERAL but is blocked by STRICT is known (e.g., a prompt with a restricted-domain term).

**Steps:**
1. Submit the prompt with profile `GENERAL`. Note the result.
2. Submit the same prompt with profile `CONSERVATIVE`. Note the result.
3. Submit the same prompt with profile `STRICT`. Note the result.

**Expected:**
- Under `GENERAL`: card reaches `REPLY_RECORDED`. Input guardrail PASSED.
- Under `CONSERVATIVE`: result depends on whether the prompt matches any restricted-domain term. Typically `REPLY_RECORDED` for a mild restricted-domain prompt; the blocked path requires a term explicitly in the CONSERVATIVE vocabulary.
- Under `STRICT`: card reaches `BLOCKED`. The `ruleId` identifies the matched pattern.
- All three interactions are visible in the live list. Status pills clearly distinguish the outcomes.

---

## J6 — SSE stream delivers all state transitions without page refresh

**Preconditions:**
- Service is running.
- The browser's App UI tab is open with the SSE connection active (no page refresh since tab open).

**Steps:**
1. Submit a clean prompt under GENERAL.
2. Observe the live list without refreshing the page.

**Expected:**
- The card transitions through `SUBMITTED` → `PROMPT_CHECKED` → `REPLYING` → `REPLY_RECORDED` live, each transition driven by an SSE event.
- No polling is visible in the browser's network tab — only the long-lived `/api/interactions/sse` connection and the initial POST.
- After `REPLY_RECORDED`, the detail pane updates immediately on the right side when the card is selected, without a separate API call to fetch the full interaction.
