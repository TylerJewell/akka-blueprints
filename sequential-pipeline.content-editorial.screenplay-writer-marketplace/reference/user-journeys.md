# User journeys — screenplay-writer-marketplace

## J1 — Submit a seeded email thread and receive a delivered screenplay

**Preconditions:** Service running on declared port (`http://localhost:9817/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded source `Merger announcement email thread` has a matching fixture in `src/main/resources/sample-data/sources/`.

**Steps:**
1. Open `http://localhost:9817/` → App UI tab.
2. From the **Pick a seeded source** dropdown, pick `Merger announcement email thread`.
3. Click **Write screenplay**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `PARSING` within 1 s more.
- Within ~20 s the card reaches `PARSED`. The right pane shows the Parsed-source table with ≥ 2 characters, ≥ 1 setting, and ≥ 2 beats; all character entries use `[NAME_N]` placeholders, not raw names.
- Within ~20 s more the card reaches `DEVELOPED`. The right pane shows ≥ 2 scenes with correct sluglines and beat assignments.
- Within ~20 s more the card reaches `FORMATTED`, then `DELIVERED` within 1 s. The right pane shows a `Screenplay` with `scenes.length == scenePlan.scenes.length`, all action lines use placeholder names, and the delivery-check chip shows PASSED.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — PII sanitizer replaces email addresses and names before agent ingestion

**Preconditions:** Service running. The submitted source text contains a real email address (`alice@example.com`) and a display-name header (`From: Alice Smith <alice@example.com>`).

**Steps:**
1. Paste source text containing `alice@example.com` and `Alice Smith` into the text area.
2. Click **Write screenplay**.
3. Wait for `DELIVERED`.
4. Inspect the row via `GET /api/screenplays/{id}`.

**Expected:**
- The `parsedSource.characters` list contains an entry with `placeholder = "[NAME_1]"` and `placeholder = "[EMAIL_1]"` — no raw `Alice Smith` or `alice@example.com` appears in any lifecycle field.
- The `screenplay.scenes` action lines reference `[NAME_1]`, not `Alice Smith`.
- The `deliveryCheck.passed` field is `true` and `detectedTokens` is empty.
- The service log shows `PiiSanitizer: replaced 2 tokens` at the point of ingestion, before any entity write.

## J3 — Delivery guardrail blocks a screenplay containing residual PII

**Preconditions:** Mock LLM mode. The mock's `format-screenplay.json` includes one entry whose first `SceneBlock.action` field contains the raw email address `alice@example.com` (simulating a case where the FORMAT task reconstructed a PII value not present in the sanitized input).

**Steps:**
1. Submit the seeded source that maps to the hallucinated-PII mock entry (determined by `MockModelProvider.seedFor(screenplayId)` — occurs on the second submission in a sequence of six).
2. Watch the live list for a card whose delivery-check chip shows BLOCKED.

**Expected:**
- The screenplay reaches `FORMATTED` (the FORMAT task completed), then immediately transitions to `DELIVERY_BLOCKED`.
- The `deliveryCheck.passed` field is `false` and `detectedTokens = ["alice@example.com"]`.
- The delivery-block log strip on the card shows the detected token, the reason string (`pii-detected: alice@example.com found in formatted output`), and the timestamp.
- The raw email address does not appear anywhere else in the record — the entity write of `ScreenplayFormatted` preceded the guardrail check; the screenplay payload itself is stored but the entity's status is `DELIVERY_BLOCKED`.
- The other submissions in the sequence are DELIVERED with PASSED chips.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded source.
2. Wait for `DELIVERED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the screenplayId.

**Expected:**
- The PARSE task's log entries show only `extractCharacters`, `extractSettings`, and `extractBeats` calls.
- The DEVELOP task's log entries show only `buildSceneOutline` and `assignBeatsToScenes` calls.
- The FORMAT task's log entries show only `formatSlugline`, `formatAction`, and `formatDialogue` calls.
- No cross-phase calls appear.
- The order of tasks in the log is PARSE → DEVELOP → FORMAT. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Scene-outline parity between ScenePlan and Screenplay

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded source `Product launch campaign emails` (whose mock-paired `ScenePlan` carries 4 scenes).
2. Wait for `DELIVERED`.

**Expected:**
- The recorded `Screenplay.scenes.length` equals the recorded `ScenePlan.scenes.length` (both 4).
- Every `SceneBlock.sceneId` matches one `SceneOutline.sceneId` from the plan, one-to-one.
- Every `SceneBlock.slugline` equals the corresponding `SceneOutline.slugline` exactly.

## J6 — Empty source text produces an empty screenplay without crashing

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Type a source title but leave the source text field blank (or submit whitespace only).
2. Click **Write screenplay**.

**Expected:**
- `PiiSanitizer` runs on the empty text and returns a `SanitizedSource` with `sanitizedText = ""` and an empty placeholder map.
- The PARSE task returns a `ParsedSource` with `characters = []`, `settings = []`, `beats = []`.
- The DEVELOP task returns a `ScenePlan` with `scenes = []`.
- The FORMAT task returns a `Screenplay` with `title = "(no scenes)"`, `logline = "(no source material)"`, and `scenes = []`.
- The delivery guardrail passes (no PII tokens to check against; output is empty).
- The card reaches `DELIVERED` with a PASSED chip. Nothing crashes; the empty screenplay is honestly empty.
