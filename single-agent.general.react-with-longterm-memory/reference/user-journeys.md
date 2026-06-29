# User journeys — mem0-react-agent

## J1 — Store a preference and recall it in a later session

**Preconditions:** Service running on declared port (`http://localhost:9621/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9621/` → App UI tab.
2. In the userId field enter `user-alice`. Type "I prefer temperatures in Fahrenheit." and click **Send**.
3. Wait for the agent's answer to appear. Observe the Tool calls accordion inside the agent bubble.
4. Click **New session** to start a fresh conversation (same userId `user-alice`).
5. Type "What is a comfortable room temperature?" and click **Send**.

**Expected:**
- In step 2, the agent's turn shows at least one `store-memory` tool call with input text containing "Fahrenheit".
- Within ~1 s of the turn completing, a new row appears in the right memory panel with `cleanText` containing "Fahrenheit" and status `PERSISTED`.
- In step 5, the new session's agent invokes `recall-memories` first (visible in tool calls), receives the stored fact, and answers with a Fahrenheit value without the user mentioning the preference again.

## J2 — PII is redacted before any fact is persisted

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Type "My email is alice@example.com and I like espresso." as the first message for userId `user-bob`.
2. Wait for `PERSISTED` status on the stored fact.
3. Inspect the memory panel row for the espresso or email-related fact.
4. Fetch `GET /api/memory/user-bob` from the browser console.

**Expected:**
- The memory panel shows `cleanText` containing `[REDACTED-EMAIL]` (or equivalent redaction marker) and a `piiCategoriesFound` chip showing `email`. The raw string `alice@example.com` does not appear anywhere in the UI.
- The JSON from `GET /api/memory/user-bob` shows `cleanText` with the redacted form; `rawText` is absent from the MemoryRow schema (the raw field is not surfaced on the view).
- The service log (if `LOG_LEVEL=DEBUG`) shows the sanitizer found the `email` category and replaced it before the persist step ran.

## J3 — Drift monitor fires when fact count crosses threshold

**Preconditions:** Service running with mock LLM. `drift-threshold` temporarily lowered to 3 in `application.conf` for the test (or achieved by submitting enough turns).

**Steps:**
1. Start three separate sessions for userId `user-charlie`, each sending a message that causes the agent to store a fact (e.g., three distinct preferences).
2. After the third fact reaches `PERSISTED`, observe the right memory panel.

**Expected:**
- A warning banner appears in the drift monitor strip: "Memory drift detected for user-charlie: 3 facts (threshold: 3)."
- The banner includes `signaledAt` timestamp.
- The agent's loop is unaffected — the fourth session (if started) continues normally.
- The `/api/memory/sse` stream delivered a `drift-signal` event matching the banner content.

## J4 — Multi-step reasoning uses the calculate tool

**Preconditions:** Service running. Any model provider.

**Steps:**
1. For userId `user-dave`, type "If I drive 120 km at 80 km/h and then 50 km at 100 km/h, what is my total travel time in minutes?"
2. Wait for the agent's answer.

**Expected:**
- The agent's tool calls accordion shows at least one `calculate` invocation with an arithmetic expression (e.g., `120/80 + 50/100` or equivalent steps).
- The answer contains the correct total (approximately 120 minutes).
- The agent does NOT store a memory fact for this turn — the question is ephemeral arithmetic, not a personal preference or stable user context.

## J5 — Session history preserves earlier turns

**Preconditions:** Service running. Any model provider.

**Steps:**
1. For userId `user-eve`, send two messages in the same session: first "My name is Eve." then "What did I just tell you?"
2. Wait for both turns to complete.

**Expected:**
- The first turn's agent answer acknowledges the name.
- The second turn's agent answer recalls "Eve" from the session context (not from long-term memory — within-session context is available in the task instructions). The `recall-memories` tool call returns any pre-existing facts, but the name need not have been persisted yet for the agent to know it within the session.
- `GET /api/sessions/{id}` returns a `turns` array with two entries.

## J6 — Failed sanitizer does not persist raw text

**Preconditions:** Service running with a mock that returns a sanitizer error for a specific pattern (achievable by injecting a test fact via a direct `MemoryEntity.requestStore` call, or by triggering the error branch in the mock sanitizer).

**Steps:**
1. Trigger the error branch in `FactSanitizer` (implementation-dependent — see the sanitizer's error-injection test hook in the generated code).
2. Observe `MemoryEntity` status.

**Expected:**
- The fact transitions to `FAILED` status.
- No `FactPersisted` event is emitted.
- The `MemoryView` facts list does not contain the raw text.
- `MemoryWriteWorkflow` does not advance past `awaitSanitizedStep` (the sanitizer never calls `attachSanitized`), and the workflow's error step transitions the entity to `FAILED`.
