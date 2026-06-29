# User journeys — bug-assistant

Numbered acceptance journeys. Each defines the precondition, the steps a tester follows, and the expected outcome. The generated system must pass all four.

---

## J1 — Happy-path resolution (Java NullPointerException)

**Precondition:** Service is running. No bugs in the system.

**Steps:**
1. Open the App UI tab.
2. Select the "Java NPE" seeded example from the dropdown. Title and description fields populate automatically.
3. Set priority to `HIGH`, component to `user-service`. Click **Submit bug**.
4. Observe the live list. A new card appears with status `SUBMITTED`.
5. Within 2 s, the card transitions to `ENRICHED`. The right-side detail shows ticket key `PROJ-NNNN`, assignee, and labels.
6. The card transitions to `INVESTIGATING`. The tool-call trace begins populating with `read_ticket` and `search_web` entries.
7. Within 30 s, the card transitions to `RESOLVED`. The resolution badge shows `FIXED`. The resolution body is at least two sentences. `confidenceLevel` is `HIGH` or `MEDIUM`. The tool-call trace shows at least one `search_web` call and one `write_ticket` call, both with `PASSED` outcome.

**Expected:** `status = RESOLVED`, `resolution.status = FIXED`, `resolution.resolutionBody` ≥ 20 characters, tool-call trace contains `write_ticket` with outcome `PASSED`.

---

## J2 — Guardrail blocks a bad write, agent corrects and retries

**Precondition:** Service is running. Mock LLM is configured to return a malformed `write_ticket` call (empty `resolutionBody`) on the first iteration of every third bug.

**Steps:**
1. Submit bugs until the third one triggers the mock's malformed path. (Alternatively: trigger directly by submitting three bugs in sequence with the seeded "Go deadlock" example.)
2. Observe the tool-call trace in the right-side detail for the third bug.
3. Confirm the first `write_ticket` entry has outcome `REJECTED` and the guardrail error reads: "resolutionBody must be at least 20 characters."
4. Confirm the agent retries: a second `write_ticket` entry appears with outcome `PASSED` and a non-empty resolution body.
5. Confirm the bug card eventually reaches `RESOLVED` status with a valid resolution body visible.

**Expected:** Tool-call trace contains exactly two `write_ticket` entries for this bug — first `REJECTED`, second `PASSED`. Final `status = RESOLVED`. The rejected call's arguments are not persisted as the resolution.

---

## J3 — Low-confidence resolution when search results are empty

**Precondition:** Service is running. The mock search handler is configured to return empty snippets for the "Python async race condition" seeded bug.

**Steps:**
1. Select the "Python async race" seeded example. Submit with any priority.
2. Wait for `RESOLVED` status.
3. Check the resolution detail. `confidenceLevel` chip is `LOW`.
4. Check `resolutionBody` — it contains language indicating no matching search context was found (e.g., "No directly relevant results were found; the following is based on general async patterns…").
5. Confirm the bug is not `FAILED` — the agent produced a resolution regardless.

**Expected:** `status = RESOLVED`, `resolution.confidenceLevel = LOW`, `resolution.resolutionBody` non-empty and references the absence of search results. `resolution.evidence` list is empty or contains only empty-snippet entries.

---

## J4 — SSE stream delivers all transitions; late-joining client receives full current row

**Precondition:** Service is running.

**Steps:**
1. Open a terminal. Connect to the SSE endpoint: `curl -N http://localhost:9702/api/bugs/sse`.
2. Submit a bug via the UI.
3. Confirm the terminal receives four `bug-update` events in order: `SUBMITTED`, `ENRICHED`, `INVESTIGATING`, `RESOLVED`.
4. After the bug reaches `RESOLVED`, open a second terminal. Connect to the same SSE endpoint.
5. Submit a second bug via the UI.
6. Confirm the second terminal receives all transitions for the second bug starting from `SUBMITTED`.
7. Disconnect both terminals. Reconnect a single terminal while no bug is in progress.
8. Submit a third bug. Confirm the reconnected terminal receives all transitions for the third bug without missing any.

**Expected:** Every state transition emits exactly one SSE event. Each event payload includes the full `Bug` row at that moment (not a delta). A browser connecting mid-workflow receives the current full row on its next incoming event.
