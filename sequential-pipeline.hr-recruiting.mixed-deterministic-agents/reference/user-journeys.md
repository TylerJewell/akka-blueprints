# User journeys — score-aggregator

## J1 — Submit a seeded candidate and receive a completed recommendation

**Preconditions:** Service running on declared port (`http://localhost:9696/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded candidate `Alice Chen (Staff Engineer)` has a matching `src/main/resources/sample-data/history/alice-chen.json` and the role `src/main/resources/sample-data/roles/staff-software-engineer.json` both exist.

**Steps:**
1. Open `http://localhost:9696/` → App UI tab.
2. From the **Pick a seeded candidate** shortcut, pick `Alice Chen (Staff Engineer)`.
3. Click **Submit application**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `SCREENING` within 1 s more.
- Within ~20 s the card reaches `SCREENED`. The right pane shows the Screen-result panel with ≥ 2 skill-match rows and a non-empty fit summary.
- Within ~1 s the card reaches `SCORED`. The Candidate-score panel shows five dimension rows and a total score in [0, 100]. The pipeline-phase timeline shows SCORE lighting up in the Java (deterministic) colour.
- Within ~20 s more the card reaches `RECOMMENDED`. The recommendation badge shows `ADVANCE`, `HOLD`, or `REJECT` with a one-sentence rationale.
- Within ~1 s the card reaches `NOTIFIED` and the gate result chip shows green `PASS`. The Status-update panel shows a non-empty notification payload.
- Total elapsed time: ≤ 60 s.

## J2 — CI test gate blocks a build with a defective ScoreAggregator

**Preconditions:** Development environment. The `ScoreAggregatorTest` fixture includes a test that forces `ScoreAggregator` to return `totalScore = 150` (by passing an uncapped inputs fixture).

**Steps:**
1. Edit `ScoreAggregator.java` to remove the `Math.min(total, 100)` cap (simulate a defect).
2. Run `/akka:build`.

**Expected:**
- `ScoreAggregatorTest` fails during the Maven `test` phase with an assertion error: `Expected totalScore ≤ 100, got 150`.
- The build exits with a non-zero code. The service does not start.
- No `http://localhost:9696/` is reachable.
- Restoring the `Math.min` cap makes the test pass and the service start normally.

## J3 — QualityGate flags a runtime score with a missing dimension

**Preconditions:** Mock LLM mode. A mock `ScreenResult` fixture with `passedInitialScreen = false` is injected so `ScoreAggregator` skips the "screen-flag" dimension (simulated by a test path that omits that dimension key).

**Steps:**
1. Submit any seeded candidate.
2. Watch the live list until the gate result chip appears.

**Expected:**
- The application reaches `NOTIFIED`.
- The gate result chip shows orange `FAIL`.
- The `GateResult.reason` reads: `"DimensionScore count was 4; expected 5."` (or equivalent naming the missing dimension key).
- The card border highlights orange. The recruiter sees the annotation and knows to inspect before advancing the candidate.
- The `StatusUpdate` is still recorded (gate is non-blocking — it annotates, it does not abort).

## J4 — Pipeline step log shows LLM vs. deterministic split

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider (real or mock).

**Steps:**
1. Submit any seeded candidate.
2. Wait for `NOTIFIED`.
3. Inspect the service log for step-boundary lines. Filter to entries tagged with the applicationId.

**Expected:**
- `screenStep` log shows `workflow.step.start(screenStep)` followed by `agent.task.call(SCREEN_RESUME)` and tool calls (`lookupRoleRequirements`, `checkSkillMatch`). No direct Java method calls appear — the LLM ran this step.
- `scoreStep` log shows `workflow.step.start(scoreStep)` followed by a direct `ScoreAggregator.score(...)` invocation log line. No `agent.task.call` line appears — no LLM ran this step.
- `recommendStep` log shows `workflow.step.start(recommendStep)` followed by `agent.task.call(GENERATE_RECOMMENDATION)` and tool calls (`fetchCandidateHistory`, `lookupMarketBenchmark`). LLM step.
- `notifyStep` log shows `workflow.step.start(notifyStep)` followed by direct `StatusNotifier.notify(...)` and `QualityGate.evaluate(...)` invocation lines. No `agent.task.call` line — deterministic step.
- The order in the log is: screenStep → scoreStep → recommendStep → notifyStep.

## J5 — REJECT recommendation with low score

**Preconditions:** Service running. Mock LLM mode. The mock `generate-recommendation.json` includes one entry with `decision: REJECT`.

**Steps:**
1. Submit seeded candidate `Bob Martins (Product Manager)` for `Staff Software Engineer` (a deliberate mismatch).
2. Wait for `NOTIFIED`.

**Expected:**
- The `CandidateScore.totalScore` is ≤ 40 (low skills-match, low experience-years for a staff engineering role).
- The recommendation badge shows `REJECT` with a rationale citing the low score.
- The `StatusUpdate.decision` is `REJECT`.
- The gate result chip shows green `PASS` (the gate checks score validity, not the decision — a valid low score is a valid score).

## J6 — Custom candidate with no history file

**Preconditions:** Service running. No `src/main/resources/sample-data/history/<candidate-slug>.json` exists for the custom candidate.

**Steps:**
1. In the App UI, fill in a custom candidate name and resume text. Select any target role.
2. Click **Submit application**.

**Expected:**
- `screenStep` completes normally — `lookupRoleRequirements` returns the role file, `checkSkillMatch` assesses the provided skills.
- `scoreStep` completes normally — `ScoreAggregator` runs on the `ScreenResult` with no dependency on history.
- `recommendStep`: `RecommendTools.fetchCandidateHistory` returns an empty `CandidateHistory` (no file found). `lookupMarketBenchmark` returns the benchmark for the selected role. `ScreeningAgent` produces a `Recommendation` based on available data.
- The application reaches `NOTIFIED` and the gate result chip shows `PASS`.
- Nothing crashes; the pipeline handles absent history gracefully.
