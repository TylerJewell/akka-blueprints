# User journeys — vto-genmedia

## J1 — Submit a garment try-on and receive evaluated media

**Preconditions:** Service running on declared port (`http://localhost:9809/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded garment `summer-dress-blue` has a matching `src/main/resources/sample-data/garments/summer-dress-blue.json` file and a corresponding generated asset in `src/main/resources/sample-data/generated/`.

**Steps:**
1. Open `http://localhost:9809/` → App UI tab.
2. From the **Garment** dropdown, select `Summer Dress — Ocean Blue`.
3. Set **Model preset** to `standing-front`.
4. Check **Include video clip**.
5. Click **Run try-on**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `PREPARING` within 1 s more.
- Within ~15 s the card reaches `ASSETS_READY`. The right pane shows the Asset summary panel with garment metadata and preset details; primary colour swatch is `#1E90FF`.
- Within ~30 s the card reaches `MEDIA_GENERATED`. The right pane shows the Generated media panel with a composite image thumbnail and a video clip player placeholder. `generationParams` shows the model and background colour.
- Within ~15 s more the card reaches `VALIDATED`, then `EVALUATED` within 1 s of that. The right pane shows both Dimension (`aspectRatioOk: true`) and Colour fidelity (`colourFidelityOk: true`, `deltaE ≤ 10`) badges green. Safety verdict badge shows `CLEARED`. Quality score chip shows 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Safety guardrail blocks an unsafe generated composite

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `generate-media.json` includes one entry whose `composite.url` contains the substring `unsafe-fixture`.

**Steps:**
1. Submit the seeded garment `summer-dress-blue` with preset `standing-front` three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/tryons/sse`).

**Expected:**
- On the third submission's `generateStep`, the agent's first LLM response produces a `MediaResult` whose `composite.url` contains `unsafe-fixture`. `ImageSafetyGuardrail` rejects the output; a `SafetyRejected{category: "EXPLICIT", reason: "safety-violation: EXPLICIT — composite url contains 'unsafe-fixture'", rejectedAt: ...}` event lands on the entity.
- The rejected `MediaResult` is never written to `TryOnEntity` — there is no `MediaGenerated` event following the rejection.
- The agent's second iteration produces a clean `MediaResult`. The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The safety-rejection log strip on the right pane shows the one rejected output with its full structured reason and category.

## J3 — Colour fidelity gap drives a low quality score

**Preconditions:** Mock LLM mode. The mock's `validate-output.json` includes one entry whose `ColourReport.deltaE > 10.0` (colour fidelity failure).

**Steps:**
1. Submit the seeded garment `linen-trousers-beige` (whose mock-paired `ValidatedMedia` carries the high-deltaE `ColourReport`) with any preset.
2. Wait for the card to reach `EVALUATED`.

**Expected:**
- The try-on request completes (the safety guardrail only checks for unsafe content, not colour fidelity).
- The validated media's Colour fidelity badge in the right pane shows a red cross with `deltaE > 10.0` and the `dominantActual` vs `expectedPrimary` swatches.
- The quality score chip shows **3** or lower; the rationale reads: `"Colour fidelity failed: deltaE <value> exceeds threshold 10.0."`.
- The card border highlights red (score ≤ 2 triggers this; score 3 shows an amber border instead — the exact cutoff is at the implementer's discretion within ≤ 2 = red).
- The other garments in the same session score ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider.

**Steps:**
1. Submit the seeded garment `polo-shirt-white` with preset `seated` and `includeVideo = false`.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the `tryOnId`.

**Expected:**
- The PREPARE task's log entries show only `resolveGarment` and `resolveModelPreset` calls.
- The GENERATE task's log entries show `compositeImage` and — because `includeVideo = false` — no `renderVideoClip` call.
- The VALIDATE task's log entries show only `checkAspectRatio` and `checkColourFidelity` calls.
- No cross-phase calls appear. The order of tasks in the log is PREPARE → GENERATE → VALIDATE. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Video clip omitted when includeVideo is false

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit any seeded garment with `includeVideo = false`.
2. Wait for `EVALUATED`.

**Expected:**
- The `TryOnRecord.media.videoClip` field is `null` on the wire.
- The Generated media panel in the UI shows only the composite image thumbnail; the video clip player area is absent.
- The quality score is 4 (the "video clip present when requested" rule scores 1 point; since `includeVideo = false`, that rule is not checked and the score is based on the remaining three checks).
- The rationale reads: `"Aspect ratio, colour fidelity, and composite presence satisfied; video not requested."` (or equivalent wording from the scorer).

## J6 — Unknown garment ID returns a placeholder composite

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/garments/<custom-id>.json` exists.

**Steps:**
1. POST directly to `/api/tryons` with body `{"garmentId": "mystery-jacket-plaid", "modelPreset": "standing-front", "includeVideo": false}`.
2. Wait for `EVALUATED`.

**Expected:**
- `PrepareTools.resolveGarment("mystery-jacket-plaid")` returns a `GarmentSpec` with `garmentId = "mystery-jacket-plaid"`, `displayName = "(unknown garment)"`, and empty or placeholder `imageUrl`.
- The agent's GENERATE task returns a `MediaResult` with `composite.url = "placeholder://no-garment"`.
- The safety guardrail passes (the placeholder URL does not contain `unsafe-fixture`).
- The quality score is 1 or 2 (composite URL is technically present but placeholder; colour fidelity cannot be computed against an empty `primaryColour`, so that rule fails).
- Nothing crashes; the empty/placeholder try-on record is honest about the gap.
