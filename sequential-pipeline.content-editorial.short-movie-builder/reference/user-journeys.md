# User journeys — short-movie-agents

## J1 — Submit a creative brief and get a movie package

**Preconditions:** Service running on declared port (`http://localhost:9732/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded brief `a 60-second workplace-safety training video` has a matching `src/main/resources/sample-data/briefs/workplace-safety-training.json` file.

**Steps:**
1. Open `http://localhost:9732/` → App UI tab.
2. From the **Pick a seeded brief** dropdown, pick `a 60-second workplace-safety training video`.
3. Click **Produce movie**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `SCRIPTING` within 1 s more.
- Within ~20 s the card reaches `SCRIPTED`. The right pane shows the Script panel with ≥ 3 rows; each row has a non-empty sceneId, type, description, and dialogueLine.
- Within ~20 s more the card reaches `STORYBOARDED`. The right pane shows a shot list with one shot per scene; every `shot.sceneId` matches a `scene.sceneId` from the script panel.
- Within ~20 s more the card reaches `ASSEMBLED`. The right pane shows the package-scene table with one row per shot in ascending orderIndex, and a total-runtime badge > 0.
- Within ~20 s more the card reaches `REVIEWED`. The coherence-check table shows all scenes marked coherent, the review summary reads "All scenes coherent.", and the coherence score chip shows 5/5.
- Total elapsed time: ≤ 80 s on the happy path.

## J2 — Content-safety guardrail blocks a policy-violating script

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `write-script.json` includes one entry whose script text contains a regulated brand name used as a fictional entity — this is the deliberately policy-violating entry.

**Steps:**
1. Submit any seeded brief three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/productions/sse`).

**Expected:**
- On the third submission's `scriptStep`, the agent's first iteration returns a script containing a regulated brand name. `ContentSafetyGuard` rejects the result; a `SafetyBlockRecorded{phase: "SCRIPT", violationType: "regulated-brand", reason: "..."}` event lands on the entity.
- The policy-violating script NEVER lands on `MovieEntity` — there is no `ScriptWritten` event for that iteration.
- The agent's second iteration produces a clean script. The card eventually reaches `REVIEWED` as in J1.
- The card in the App UI shows the small orange dot indicating a safety block fired. The safety-block log strip on the right pane shows the one blocked response with its phase, violationType, reason, and timestamp.

## J3 — Broken shot reference flags coherence score 1

**Preconditions:** Mock LLM mode. The mock's `assemble-package.json` includes one entry whose first `PackageScene` cites a `shotId` absent from the paired `Storyboard`.

**Steps:**
1. Submit any seeded brief six times. (The broken-reference entry is selected once in every six runs by the mock's `seedFor(productionId)` modulo.)
2. Watch the live list until a production's card border highlights red.

**Expected:**
- The flagged production reaches `REVIEWED` (the safety guard only checks content policy, not shot references).
- The coherence score chip shows **1** (shot reference validity check failed; the other three rules may or may not pass). The rationale reads: *"Shot reference validity failed: packageScene 'ps-001' references shotId '...' which does not appear in the Storyboard."*
- The card's border highlights red. The producer knows to inspect this package before sending to rendering.
- The other five productions in the run scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded brief.
2. Wait for `REVIEWED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the productionId.

**Expected:**
- The SCRIPT task's log entries show only `generateScenes` and `writeDialogueLine` calls.
- The STORYBOARD task's log entries show only `planShot` and `selectFraming` calls.
- The ASSEMBLE task's log entries show only `buildPackageScene` and `computeRuntime` calls.
- The REVIEW task's log entries show only `checkSceneCoherence` and `generateReviewSummary` calls.
- No cross-phase tool calls appear.
- The order of tasks in the log is SCRIPT → STORYBOARD → ASSEMBLE → REVIEW. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Scene-shot parity across all phases

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded brief `a nature documentary short about urban foxes` (whose mock-paired `MovieScript` carries 5 scenes).
2. Wait for `REVIEWED`.

**Expected:**
- The recorded `Storyboard.shots` length equals the recorded `MovieScript.scenes` length (both 5).
- The recorded `AssembledPackage.packageScenes` length equals 5.
- Every `packageScene.shotId` appears in `storyboard.shots` and every `packageScene.sceneId` appears in `movieScript.scenes`.
- The coherence score chip shows 5/5 with rationale "Scene coverage, shot reference validity, runtime plausibility, and order consistency all satisfied."

## J6 — Custom brief with no matching sample-data file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/briefs/<custom-slug>.json` exists for the user's brief.

**Steps:**
1. In the App UI, type a custom brief (e.g. `a short film about competitive cheese rolling`) into the brief field.
2. Click **Produce movie**.

**Expected:**
- `ScriptTools.generateScenes` returns an empty list.
- The agent's SCRIPT task returns a `MovieScript` with `scenes = []`.
- The workflow advances to `storyboardStep`. The STORYBOARD task returns a `Storyboard` with `shots = []`.
- The workflow advances to `assembleStep`. The ASSEMBLE task returns an `AssembledPackage` with `packageScenes = []` and `totalRuntimeSeconds = 0`.
- The workflow advances to `reviewStep`. The REVIEW task returns a `ReviewResult` with an empty `coherenceChecks` list and `summary = "No scenes to review."`.
- The coherence score chip shows 1 (runtime plausibility fails at 0; no scenes to cover). The rationale names "runtime plausibility: totalRuntimeSeconds is 0."
- The pipeline completes; nothing crashes; the empty package is honestly empty.
