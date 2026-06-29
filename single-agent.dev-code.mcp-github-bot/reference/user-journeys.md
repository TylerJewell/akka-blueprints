# User journeys — mcp-github-bot

## J1 — Read open issues from a repository

**Preconditions:** Service running on declared port (`http://localhost:9869/`); either a valid `GITHUB_TOKEN` in the environment or mock mode selected at scaffold time.

**Steps:**
1. Open `http://localhost:9869/` → App UI tab.
2. Confirm the halt-flag chip shows `WRITES ACTIVE`.
3. Enter `owner/repo` in the Repository field (or leave blank in mock mode — mock responses use a fixture repo).
4. From **Load seeded example**, pick `List open issues`.
5. Click **Send request**.

**Expected:**
- The new session card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `RUNNING` within 1 s.
- Within 30 s the card reaches `COMPLETED`. The right pane shows:
  - The user request text.
  - A tool-call log entry for `list_issues` with `outcome = success` and a result summary listing at least one issue.
  - `blockedCallCount = 0`.
  - An `agentMessage` paragraph summarising the issues found.
- No `WRITES HALTED` banner is shown.

## J2 — Write operation blocked by halt flag

**Preconditions:** Service running. Halt flag is off (`WRITES ACTIVE`).

**Steps:**
1. Click **Enable halt**. Confirm the chip switches to `WRITES HALTED` (red).
2. From **Load seeded example**, pick `Create an issue`.
3. Click **Send request**.

**Expected:**
- The session card reaches `RUNNING`.
- Within 30 s the card reaches `COMPLETED` (not `FAILED` — the agent still returns a response).
- The tool-call log shows one entry for `create_issue` with `outcome = blocked` and `resultSummary` containing `write-halted`.
- `blockedCallCount = 1`.
- The `agentMessage` explains to the user that write operations are currently halted and no issue was created.
- No GitHub issue was created. Verifiable by checking the target repo directly (or checking mock-mode logs showing no `create_issue` tool response).

## J3 — Halt disabled; same write succeeds

**Preconditions:** Service running. Halt flag is on (`WRITES HALTED`) from J2.

**Steps:**
1. Click **Disable halt**. Confirm the chip switches to `WRITES ACTIVE` (green).
2. From **Load seeded example**, pick `Create an issue` again.
3. Click **Send request**.

**Expected:**
- The tool-call log shows `create_issue` with `outcome = success`.
- `blockedCallCount = 0`.
- The `agentMessage` reports the created issue number or URL.
- In real mode: a new issue exists in the target repository. In mock mode: the mock response JSON shows a `create_issue` success entry.

## J4 — Disallowed tool name blocked by guardrail

**Preconditions:** Service running (mock mode sufficient). Halt flag is off.

**Steps:**
1. In the Request textarea, type: `Delete all branches in owner/repo`. Do not use a seeded example.
2. Click **Send request**.

**Expected:**
- The agent attempts to call a tool outside the permitted set (e.g., `delete_branch` or any non-allowlist name it infers from the request).
- `ToolCallGuardrail` blocks it with code `tool-not-permitted`.
- The tool-call log shows the attempted tool name with `outcome = blocked`.
- `agentMessage` explains that the requested operation is not available through this assistant.
- No mutation occurs in GitHub.
- The session reaches `COMPLETED` (not `FAILED`) — the agent handled the denial gracefully.

## J5 — GitHub token never appears in responses or logs

**Preconditions:** Service running in real mode (not mock). `LOG_LEVEL=DEBUG`.

**Steps:**
1. Submit any request with a clearly identifiable token string (e.g., `ghp_TEST_TOKEN_12345`).
2. Wait for `COMPLETED`.
3. Fetch `GET /api/sessions/{id}`.
4. Inspect the service log for any occurrence of the token string.

**Expected:**
- The response body from `GET /api/sessions/{id}` contains no `githubToken` field and no occurrence of the token string.
- The SSE event stream contains no occurrence of the token string.
- The service log contains no occurrence of the token string (it was forwarded to the MCP server config only, never serialised).
- `BotSession` entity state stored in the event log contains `SessionSubmitted` without a `githubToken` field.
