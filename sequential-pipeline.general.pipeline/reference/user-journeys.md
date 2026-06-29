# User journeys — pipeline

## J1 — Submit a topic and get a briefing

**Preconditions:** Service running on declared port (`http://localhost:9652/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded topic `LLM token-cost trends` has a matching `src/main/resources/sample-data/signals/llm-token-cost-trends.json` file.

**Steps:**
1. Open `http://localhost:9652/` → App UI tab.
2. From the **Pick a seeded topic** dropdown, pick `LLM token-cost trends`.
3. Click **Run pipeline**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `COLLECTING` within 1 s more.
- Within ~20 s the card reaches `COLLECTED`. The right pane shows the Collected-signals table with ≥ 4 rows; each row has a non-empty source, url, and snippet.
- Within ~20 s more the card reaches `ANALYZED`. The right pane shows ≥ 2 themes and ≥ 4 claims; every `claim.supportingSource` matches a `signal.source` from the collected table.
- Within ~20 s more the card reaches `REPORTED`, then `EVALUATED` within 1 s of that. The right pane shows a `Briefing` with `sections.length == themes.length`, every section has at least one source, every source url matches a signal url, and the eval score chip shows 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Phase-gate guardrail blocks a misordered tool call

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `collect-signals.json` includes one entry whose `tool_calls` array starts with a REPORT-phase tool (`formatSection`) — this is the deliberately phase-violating entry.

**Steps:**
1. Submit any seeded topic three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/briefings/sse`).

**Expected:**
- On the third submission's `collectStep`, the agent's first iteration calls `formatSection`. `PhaseGuardrail` rejects it; a `GuardrailRejected{phase: "COLLECT", tool: "formatSection", reason: "phase-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `ReportTools.formatSection` — there is no log line from the tool body.
- The agent's second iteration falls through to a normal collect sequence (`searchSignals` + `fetchSnippet`). The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — Hallucinated source flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `write-report.json` includes one entry whose first `Section` cites a URL absent from the paired `SignalSet`.

**Steps:**
1. Submit any seeded topic six times. (The hallucinated-source entry is selected once in every six runs by the mock's `seedFor(briefingId)` modulo.)
2. Watch the live list until a briefing's card border highlights red.

**Expected:**
- The flagged briefing lands well-formed (the guardrail only checks phase order, not source provenance).
- The eval score chip shows **1** (or 2 depending on which other rules pass) and the rationale reads: *"Source provenance failed: section 'X' cites url https://example.org/... which does not appear in the recorded SignalSet."*
- The card's border highlights red. The reader knows to inspect this briefing before acting.
- The other five briefings in the run scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded topic.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the briefingId.

**Expected:**
- The COLLECT task's log entries show only `searchSignals` and `fetchSnippet` calls.
- The ANALYZE task's log entries show only `extractClaims` and `clusterClaims` calls.
- The REPORT task's log entries show only `formatSection` and `gatherSources` calls.
- No cross-phase calls appear. (If the mock-LLM violation path fires, a single `guardrail.reject` line precedes the agent's retry; the rejected call is logged but never executed.)
- The order of tasks in the log is COLLECT → ANALYZE → REPORT. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Theme-section parity

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded topic `Postgres release 17 retrospective` (whose mock-paired `Analysis` carries 4 themes).
2. Wait for `EVALUATED`.

**Expected:**
- The recorded `Briefing.sections.length` equals the recorded `Analysis.themes.length` (both 4).
- Every `section.themeId` matches one `theme.themeId` from the analysis, one-to-one.
- If the agent's first iteration produced 3 sections instead of 4 (silent collapse), the on-decision evaluator would have flagged it with a "section parity" failure and a score of 4. The presence of a score of 5 with rationale "all checks passed" confirms the parity held.

## J6 — Custom topic with no matching signal file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/signals/<custom-topic-slug>.json` exists for the user's topic.

**Steps:**
1. In the App UI, type a custom topic (e.g. `the future of artisanal pickles`) into the topic field.
2. Click **Run pipeline**.

**Expected:**
- `CollectTools.searchSignals` returns an empty list.
- The agent's COLLECT task returns a `SignalSet` with `signals = []`.
- The workflow advances to `analyzeStep`. The ANALYZE task returns an `Analysis` with `themes = []` and `claims = []` (the agent's refusal behaviour per `prompts/report-agent.md`).
- The workflow advances to `reportStep`. The REPORT task returns a `Briefing` with `title = "(no analysis themes)"` and `sections = []`.
- The eval score chip shows 1 (theme count = 0, section count = 0 — parity is satisfied, but no other rule scores). The rationale names "no themes to score."
- The pipeline completes; nothing crashes; the empty briefing is honestly empty.
