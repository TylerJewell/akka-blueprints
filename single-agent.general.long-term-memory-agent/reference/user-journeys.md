# User journeys — long-term-memory-agent

Acceptance bar for the generated system. Each journey maps to a concrete test scenario.

---

## J1 — First session: send a message, get a reply, memories are stored

**Preconditions:**
- The service is running.
- No prior sessions or memories exist for `userId = "user-demo"`.

**Steps:**
1. POST `/api/sessions` with `{ userId: "user-demo", sessionLabel: "Getting started" }` → receive `sessionId`.
2. POST `/api/sessions/{sessionId}/turns` with `{ userId: "user-demo", message: "I prefer short bullet-point answers and I'm working on a Go project called Lattice." }` → receive `turnId`.
3. Listen on `/api/sessions/{sessionId}/turns` SSE. Wait for a `turn-update` event with `status = "REPLIED"`.
4. GET `/api/memories?userId=user-demo` → expect at least one `MemoryEntry` whose `kind` is `PREFERENCE` or `CONTEXT` and whose `sanitizedText` mentions "short" or "bullet" or "Lattice".

**Expected:**
- The reply arrives within 30 s.
- `MemoryEntry` records appear within 10 s of the reply.
- No `FAILED` turn events are emitted.

---

## J2 — Second session: agent recalls facts from the first session

**Preconditions:**
- J1 has completed. At least one `MemoryEntry` for `userId = "user-demo"` exists.

**Steps:**
1. POST `/api/sessions` with `{ userId: "user-demo", sessionLabel: "Follow-up" }` → receive a new `sessionId`.
2. POST `/api/sessions/{sessionId}/turns` with `{ userId: "user-demo", message: "Can you remind me what I'm working on?" }` → receive `turnId`.
3. Wait for `turn-update` with `status = "REPLIED"` on the new session's SSE stream.
4. Read the `reply` field.

**Expected:**
- The reply references the project ("Lattice" or "Go project") or the stated preference from J1 without being told again.
- The reply does not contain raw PII strings; only sanitized-form values appear if any redaction occurred.

---

## J3 — PII in a message is sanitized before memory storage

**Preconditions:**
- The service is running. A session exists for `userId = "user-pii-test"`.

**Steps:**
1. POST `/api/sessions/{sessionId}/turns` with:
   ```json
   { "userId": "user-pii-test", "message": "My name is Carol Nguyen, my email is carol.nguyen@example.com, and my SSN is 456-78-9012. Please remember I prefer dark mode." }
   ```
2. Wait for the `REPLIED` turn event.
3. GET `/api/memories?userId=user-pii-test` → inspect all `MemoryEntry` records.

**Expected:**
- No `MemoryEntry.sanitizedText` contains `carol.nguyen@example.com`, `Carol Nguyen` (verbatim), or `456-78-9012`.
- At least one entry with `kind = PREFERENCE` references dark mode in `sanitizedText`.
- At least one entry has `piiCategoriesRedacted` non-empty, listing categories such as `"email"`, `"person-name"`, or `"ssn"`.
- The `MemoryEntity` event log (`MemoryStored` events) contains only sanitized text. The raw strings appear only in the `SessionEntity` event log (`TurnAdded.message`) — the audit trail — and are never forwarded to future model calls.

---

## J4 — Session reaches turn limit and closes

**Preconditions:**
- The service is running. A session exists for `userId = "user-limit-test"` that already has 49 turns recorded (or the developer has reduced `MAX_TURNS` to a lower value for test purposes).

**Steps:**
1. POST `/api/sessions/{sessionId}/turns` (turn 50) → expect `201 { turnId }` and a `turn-update` with `status = "REPLIED"`.
2. Listen on `/api/sessions/sse` for a `session-update` event with `status = "CLOSED"`.
3. POST `/api/sessions/{sessionId}/turns` (attempt turn 51) → expect `409 Conflict`.
4. GET `/api/sessions/{sessionId}` → `status = "CLOSED"`, `closedAt` is non-null.

**Expected:**
- The 50th turn completes normally and a reply is recorded.
- The session transitions to `CLOSED` after the 50th turn's workflow completes.
- The 51st POST returns `409` with a clear message indicating the session is closed.
- Memories extracted from turn 50 are still stored and appear in `GET /api/memories`.

---

## J5 — Agent reply references a recalled RELATIONSHIP memory

**Preconditions:**
- `userId = "user-rel-test"` has at least one stored `MemoryEntry` with `kind = RELATIONSHIP` and `sanitizedText = "User's manager is referred to as 'the Eng Director'."`.

**Steps:**
1. POST a new session for `userId = "user-rel-test"`.
2. POST a turn: `{ message: "I need to prepare a summary for my manager's review." }`.
3. Wait for `REPLIED`.

**Expected:**
- The reply incorporates the relationship context (references "Eng Director" or "your manager") without being told again in this session.
- A new `MemoryEntry` with `kind = CONTEXT` is extracted referencing the summary task.

---

## J6 — Failed turn leaves session open

**Preconditions:**
- The service is running with the mock LLM provider configured such that one turn fails (e.g., MockModelProvider returns an error for a specific userId/turnId seed).

**Steps:**
1. Trigger a turn that causes the workflow's `respondStep` to exhaust retries and call `SessionEntity.failTurn`.
2. GET `/api/sessions/{sessionId}` → inspect the turn whose `status = "FAILED"`.
3. POST another turn to the same session.

**Expected:**
- The failed turn has `status = "FAILED"` in the turns list.
- The session itself remains in `ACTIVE` status.
- The subsequent turn succeeds normally.
- No new `MemoryEntry` records are created for the failed turn.
