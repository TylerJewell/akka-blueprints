# User journeys — meeting-assistant-zoom

## J1 — Submit a seeded meeting and get a package

**Preconditions:** Service running on declared port (`http://localhost:9366/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded meeting `Q2 Engineering Retrospective` has a matching `src/main/resources/sample-data/transcripts/q2-engineering-retrospective.json` file.

**Steps:**
1. Open `http://localhost:9366/` → App UI tab.
2. From the **Pick a seeded meeting** dropdown, pick `Q2 Engineering Retrospective`.
3. Click **Process meeting**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `TRANSCRIBING` within 1 s more.
- The redaction-audit badge on the card shows a redaction count > 0 within 1 s of `TRANSCRIBING` (the sanitizer ran before the LLM call).
- Within ~25 s the card reaches `TRANSCRIBED`. The right pane shows the transcript segments table with ≥ 6 rows; every `speaker` field is a pseudonym (`[PERSON-N]`), not a real name.
- Within ~25 s more the card reaches `SUMMARIZED`. The right pane shows a summary paragraph, ≥ 2 action items with assignee pseudonyms, and ≥ 1 decision.
- Within ~25 s more the card reaches `DISPATCHED`, then `EVALUATED` within 1 s of that. The right pane shows a `MeetingPackage` with `tasks.length == actionItems.length` and `followUpEvents.length >= 1`. The eval score chip shows 5/5.
- Total elapsed time: ≤ 90 s on the happy path.

## J2 — Scope guardrail blocks a misordered tool call

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `transcribe-meeting.json` includes one entry whose `tool_calls` array starts with a DISPATCH-phase tool (`scheduleFollowUp`) — the deliberately phase-violating entry.

**Steps:**
1. Submit any seeded meeting three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/meetings/sse`).

**Expected:**
- On the third submission's `transcribeStep`, the agent's first iteration calls `scheduleFollowUp`. `ScopeGuardrail` rejects it with a phase-violation; a `ScopeRejected{phase: "TRANSCRIBE", tool: "scheduleFollowUp", reason: "phase-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `DispatchTools.scheduleFollowUp` — there is no log line from the tool body.
- The agent's second iteration falls through to a normal transcribe sequence (`segmentTranscript` + `labelSpeakers`). The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small orange dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — Write-scope check blocks dispatch to an unrecognized participant

**Preconditions:** Service running with the mock LLM selected. The mock's `dispatch-followups.json` includes one entry whose `createTask` call carries an `assigneePseudonym` of `[PERSON-99]` — a pseudonym not in the meeting's recorded participant list.

**Steps:**
1. Submit the seeded meeting `Sales Pipeline Review` six times. (The out-of-scope entry is selected once in every six runs by the mock's `seedFor(meetingId)` modulo.)
2. Watch the live list until a card shows the orange rejection dot during the `DISPATCHING` phase.

**Expected:**
- The `createTask([PERSON-99], ...)` call is rejected by `ScopeGuardrail` before the tool body executes. No `TaskRecord` is created for `[PERSON-99]`.
- A `ScopeRejected{phase: "DISPATCH", tool: "createTask", reason: "scope-violation: [PERSON-99] is not a meeting participant"}` event lands on the entity.
- The agent retries within its 4-iteration budget, this time omitting the out-of-scope pseudonym. The card eventually reaches `EVALUATED`.
- The rejection-log strip shows the one rejected call with the out-of-scope pseudonym named in the reason.

## J4 — PII never reaches the LLM

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit the seeded meeting `Product Roadmap Kickoff` (whose raw transcript includes participant names and an email address).
2. Wait for `TRANSCRIBED`.
3. Inspect the service log for lines tagged `agent.task.instructions` for the TRANSCRIBE task.

**Expected:**
- The log line for the TRANSCRIBE task's `TaskDef.instructions` contains no real participant name (e.g., no "Alice", "Bob", or the email address from the raw transcript).
- Every participant reference in the instructions is a pseudonym (`[PERSON-1]`, `[PERSON-2]`, etc.).
- The `TranscribeStarted` event on the entity carries a `redactionAudit.totalRedactions` value matching the number of distinct PII occurrences in the seeded transcript.
- The redaction-audit panel in the App UI shows the same count and lists the pattern types (`name`, `email`).

## J5 — Action-item coverage flags eval score < 5

**Preconditions:** Mock LLM mode. Any seeded meeting.

**Steps:**
1. Submit any seeded meeting where the mock's `dispatch-followups.json` entry deliberately omits a `TaskRecord` for one of the `MeetingSummary.actionItems` (the mock carries one such entry per seeded meeting).
2. Wait for `EVALUATED`.
3. Observe the eval score chip.

**Expected:**
- The `ActionItemScorer` detects that at least one `ActionItem` from `MeetingSummary.actionItems` has no corresponding `TaskRecord` in `MeetingPackage.tasks`.
- The eval score chip shows 4 (or lower if additional checks fail), and the rationale reads: *"Action-item coverage failed: action item 'ai-XXXX' has no corresponding task in the package."*
- The card border does not highlight red (score > 2), but the rationale is visible in the expanded detail.

## J6 — Empty transcript produces an honest empty package

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the App UI, paste a short informational transcript with no action items or decisions (e.g., a one-person screen-share narration).
2. Click **Process meeting**.

**Expected:**
- `PiiSanitizer` runs and produces a `RedactionAudit` (possibly zero redactions if the transcript has no PII patterns).
- The TRANSCRIBE task returns a `Transcript` with segments populated.
- The SUMMARIZE task returns a `MeetingSummary` with `actionItems = []` and `decisions = []` and a `summaryText` explaining the meeting was informational.
- The DISPATCH task returns a `MeetingPackage` with `tasks = []` and `followUpEvents = []`.
- The eval score chip shows 1 (item parity: `actionItems.size() + decisions.size() == 0`, so no items to score; rationale: "no action items or decisions to evaluate"). The pipeline completes without error; the empty package is honestly empty.
