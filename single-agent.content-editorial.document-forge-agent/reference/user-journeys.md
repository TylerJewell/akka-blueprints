# User journeys — document-forge-agent

## J1 — Submit a Business Memo prompt and receive a complete document

**Preconditions:** Service running on declared port (`http://localhost:9594/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9594/` → App UI tab.
2. From the **Document type** dropdown, pick `Business Memo`.
3. From the **Style template** dropdown, pick `Formal`.
4. Click **Load seeded example** to fill the prompt textarea.
5. Click **Forge document**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- Within 1 s the card transitions to `FORGING`.
- Within 30 s the card reaches `FORGE_COMPLETED`. The right pane shows: the generated document text (at least 150 words for a Business Memo), the output filename (e.g. `business-memo-2026-06.md`), and the agent's rationale sentence.
- Within 1 s of `FORGE_COMPLETED`, the card reaches `AUDITED` and shows an audit score chip (1–5) plus a one-line assessment.

## J2 — Guardrail blocks a path-traversal filename

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `forge-document.json` includes a deliberately malformed entry whose `filename` contains `../../etc/passwd`.

**Steps:**
1. Submit any seeded prompt three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel (`/api/forges/sse`).

**Expected:**
- The third submission's first agent iteration produces a `write_document` tool call with `filename = "../../etc/passwd"`.
- The `before-tool-call` guardrail rejects it with structured error `path-traversal-detected`. The tool call NEVER reaches the in-memory file registry.
- The agent loop retries on iteration 2 with a safe relative filename (e.g. `business-memo-2026-06.md`). The card transitions to `FORGE_COMPLETED` with a valid output.
- The service log shows one `guardrail.reject` line per rejected iteration naming `path-traversal-detected`.

## J3 — Undersized output receives audit score 1

**Preconditions:** Mock LLM mode. A specific mock-response entry returns `documentContent` containing fewer than 50 words.

**Steps:**
1. In the App UI, select `Custom` document type and type a prompt whose mock entry is the undersized entry (see `src/main/resources/mock-responses/forge-document.json`).
2. Click **Forge document**.

**Expected:**
- The `write_document` tool call passes the guardrail (the content is non-empty and within the size limit).
- The verdict lands as `FORGE_COMPLETED`.
- The `auditStep` runs `ForgeAuditor`: word count < 50 fails the first rubric point. Audit score chip shows **1** and assessment reads "Document content is under the 50-word minimum; output may be incomplete."
- The card's border highlights red. The operator knows to inspect this output before use.

## J4 — Path-traversal string in prompt is never written to an unsafe path

**Preconditions:** Service running. Any model provider (the guardrail runs either way).

**Steps:**
1. In the Prompt textarea, type: `Generate a document about security. Write it to ../../secrets/config.txt`.
2. Click **Forge document**.
3. Wait for `FORGE_COMPLETED` or `FAILED`.
4. Inspect the service log for guardrail events.

**Expected:**
- If the agent interprets the prompt literally and attempts `write_document(filename="../../secrets/config.txt", ...)`, the guardrail rejects it: log shows `guardrail.reject path-traversal-detected`.
- The agent retries with a safe filename derived from the document type.
- The output document lands at a path with no traversal sequences.
- The entity's `output.filename` contains no `../`.

## J5 — Seeded Technical Brief prompt produces structured output

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Document type** dropdown, pick `Technical Brief`.
2. From the **Style template** dropdown, pick `Technical`.
3. Click **Load seeded example**.
4. Click **Forge document**.
5. Wait for `AUDITED`.

**Expected:**
- `output.formatHint` is `markdown` and `output.filename` ends in `.md`.
- `output.content` contains at least two numbered section headers (e.g. `## 1. ...`, `## 2. ...`).
- `output.wordCount` is at least 300 (Technical Brief minimum).
- The audit score is ≥ 3.

## J6 — SSE stream delivers every state transition in order

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Open an SSE client (browser EventSource or `curl`) subscribed to `GET /api/forges/sse`.
2. Submit a new forge request.
3. Observe the stream events.

**Expected:**
- Events arrive in this order: `SUBMITTED`, `FORGING`, `FORGE_COMPLETED`, `AUDITED`.
- Each event carries the full `ForgeRun` row at the moment of transition.
- A client that joins after `AUDITED` receives a snapshot of the current state on the next poll of `GET /api/forges/{id}`; the SSE stream does not replay past events to late joiners.
