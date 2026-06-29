# User journeys — skill-patterns-tutorial

## J1 — Invoke all four patterns and read results

**Preconditions:** Service running on declared port (`http://localhost:9588/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9588/` → App UI tab.
2. Click the **Inline** card. Click **Load example** to pre-fill `Topic = "event sourcing"` and `Style = concise`. Click **Run**.
3. Wait for the COMPLETED pill. Read the output in the right pane. Verify the `wiringNote` contains "Inline skill".
4. Click the **File-based** card. Click **Load example** to pre-fill the text textarea. Click **Run**.
5. Wait for COMPLETED. Verify the output contains exactly three bullets, each ending with a parenthetical key takeaway. Verify the `wiringNote` references `summarizer-skill.md`.
6. Click the **External** card. Enter `lookupKey = "gamma"`. Click **Run**.
7. Wait for COMPLETED. Verify the output mentions "gamma" and a value from the stub. Verify the `wiringNote` references `SkillToolStub` and `lookup-tool`.
8. Click the **Meta creator** card. Enter `Skill name = "Haiku Generator"` and a short description. Click **Run**.

**Expected:**
- All four runs reach `COMPLETED` within 60 s each.
- Each run's `patternName` in the result matches the invoked card.
- Each run's `wiringNote` is distinct and describes the pattern's mechanism accurately.
- The live list shows all four runs, newest-first.

## J2 — Register a meta-created skill

**Preconditions:** J1 completed; service still running. The Meta creator run is in COMPLETED state.

**Steps:**
1. In the right pane for the Meta creator run, expand the `SkillDefinition` JSON block.
2. Verify the JSON contains `skillId`, `displayName`, `systemPrompt`, `createdBy = "SkillDemoAgent"`, and `createdAt`.
3. Click **Register**.
4. Observe the success chip on the Meta creator card.
5. Call `GET /api/skills` directly (or via browser dev tools).

**Expected:**
- `POST /api/skills` returns `201 { "skillId": "haiku-generator" }` (or the slug matching the entered name).
- `GET /api/skills` returns a list containing the newly registered `SkillDefinition`.
- A fifth card appears in the App UI grid labelled with the registered `displayName`.

## J3 — External skill tool call is logged

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`.

**Steps:**
1. Invoke the External pattern with `lookupKey = "alpha"`.
2. Wait for COMPLETED.
3. Inspect the service log.

**Expected:**
- The log contains an outbound HTTP request line for `GET http://localhost:9588/api/tool-stub/lookup?key=alpha`.
- The log contains the stub's response JSON `{ "key": "alpha", "value": "...", "source": "in-process-stub" }`.
- The result's `output` incorporates the stub value in a sentence.
- No raw API key material appears in any log line.

## J4 — Concurrent invocations complete independently

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Click **Run** on all four pattern cards in rapid succession (within 2 s of each other), using the seeded examples.
2. Monitor the live list.

**Expected:**
- All four runs appear in the live list with status `RUNNING` or `REQUESTED`.
- All four reach `COMPLETED` without any run's result appearing on the wrong card.
- The right pane shows the correct `patternName` for each selected run — none are swapped.
- No `FAILED` runs appear (assuming the LLM is reachable and the mock LLM is seeded for all four patterns).

## J5 — Unknown lookup key returns graceful output

**Preconditions:** Service running.

**Steps:**
1. Invoke the External pattern with `lookupKey = "zeta"` (not one of the five seeded keys).
2. Wait for COMPLETED.

**Expected:**
- The stub returns `{ "key": "zeta", "value": "no-data-for-key:zeta", "source": "in-process-stub" }` — HTTP 200, not 404.
- The agent incorporates the no-data response into its output (per agent prompt behavior rules for unknown keys).
- The run reaches `COMPLETED`; no `FAILED` state.
