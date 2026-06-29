# User journeys — akka-stateful-memory-agent

Six numbered acceptance journeys. Each defines the preconditions, steps, and expected outcome. These are the bar `/akka:implement` must clear.

---

## J1 — Memory persists across turns and service restarts

**Preconditions:** Service is running. No prior conversations.

**Steps:**
1. POST `/api/conversations` with `initialPersona = "You are Memo, a concise assistant."` → receive `conversationId`.
2. POST `/api/conversations/{id}/turns` with `userMessage = "I am a software engineer at a fintech startup."` → receive `turnId`.
3. Wait for `MemoryPatchApplied` SSE event on `/api/conversations/{id}/sse`.
4. GET `/api/conversations/{id}` → inspect `humanBlock.content`.
5. Restart the service.
6. POST `/api/conversations/{id}/turns` with `userMessage = "What do you remember about me?"`.
7. Wait for `AgentResponseReady` SSE event.
8. GET `/api/conversations/{id}` → inspect `history`.

**Expected:**
- After step 4, `humanBlock.content` contains a bullet mentioning "software engineer" and "fintech".
- After step 8, the agent's reply references the stored fact ("You're a software engineer at a fintech startup") without the user having to repeat it.
- `personaBlock.content` still contains "You are Memo, a concise assistant." or a superset of it.

---

## J2 — PII is redacted before reaching the memory block

**Preconditions:** Service is running. Fresh conversation.

**Steps:**
1. POST `/api/conversations` → receive `conversationId`.
2. POST `/api/conversations/{id}/turns` with `userMessage = "My email is alice@example.com and I prefer Go."`.
3. Wait for `MemoryPatchApplied` SSE event.
4. GET `/api/conversations/{id}` → inspect `humanBlock.content`.
5. POST `/api/conversations/{id}/turns` with `userMessage = "What have you learned about me?"`.
6. Wait for `AgentResponseReady`.
7. GET `/api/conversations/{id}` → inspect the agent reply in `history`.

**Expected:**
- After step 4, `humanBlock.content` contains `[REDACTED-EMAIL]` or equivalent marker, not the literal string `alice@example.com`.
- After step 7, the agent's reply does not echo the raw email address.
- The entity log's `UserMessageReceived` event still contains the original message (raw preservation for audit).

---

## J3 — Drift evaluator fires on the tenth turn

**Preconditions:** Service is running. Fresh conversation.

**Steps:**
1. POST `/api/conversations` → receive `conversationId`.
2. Send 10 POST `/api/conversations/{id}/turns` requests in sequence. Messages 1–9: each message mentions the user's nationality (e.g., "As a French person, I find…"). Message 10: any topic.
3. After the 10th turn, wait for `DriftEvaluated` SSE event.
4. GET `/api/conversations/{id}` → inspect `latestDrift`.

**Expected:**
- After step 4, `latestDrift` is not null.
- `latestDrift.riskScore` is ≥ 0.7 (nationality cluster dominates).
- `latestDrift.reason` names "nationality" as the dominant cluster.
- `latestDrift.atTurnNumber` equals 10.
- The UI drift chip on the human block is red.

---

## J4 — Persona block is never fully overwritten

**Preconditions:** Service is running. Conversation created with explicit `initialPersona`.

**Steps:**
1. POST `/api/conversations` with `initialPersona = "You are Memo. You prefer bullet points over paragraphs."` → receive `conversationId`.
2. Send 5 turns with varied topics.
3. After each turn, GET `/api/conversations/{id}` → inspect `personaBlock.content`.

**Expected:**
- After every turn, `personaBlock.content` still contains the text "You are Memo" and "bullet points".
- If the agent proposed a persona patch, it is an extension or reword of the original — not a blank replacement.
- No turn produces a `personaBlock.content` that is shorter than 20 characters.

---

## J5 — Multiple conversations are independent

**Preconditions:** Service is running.

**Steps:**
1. POST `/api/conversations` → `conversationIdA`.
2. POST `/api/conversations` → `conversationIdB`.
3. Send `"I am a data scientist."` to conversation A.
4. Send `"I am a product manager."` to conversation B.
5. Wait for both `MemoryPatchApplied` events.
6. GET `/api/conversations/{idA}` and GET `/api/conversations/{idB}`.

**Expected:**
- `humanBlock.content` for conversation A mentions "data scientist".
- `humanBlock.content` for conversation B mentions "product manager".
- Neither block contains content from the other conversation.
- GET `/api/conversations` returns both conversations.

---

## J6 — Agent produces no memory patch when nothing new is learned

**Preconditions:** Service is running. Conversation with an established human block.

**Steps:**
1. Reuse a conversation that already has a populated human block (e.g., from J1).
2. POST `/api/conversations/{id}/turns` with `userMessage = "What is 2 + 2?"`.
3. Wait for `AgentResponseReady` SSE event.
4. GET `/api/conversations/{id}` → inspect `humanBlock.lastUpdatedAt` and history.

**Expected:**
- The agent's reply answers the question ("4").
- `humanBlock.lastUpdatedAt` is unchanged from before the turn — no `MemoryPatchApplied` event was emitted.
- `history` contains the new exchange but the memory blocks are identical to their pre-turn values.
