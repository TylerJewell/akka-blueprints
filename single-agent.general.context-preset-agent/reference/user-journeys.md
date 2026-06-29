# User journeys — context-preset-agent

## J1 — prod + admin executes a privileged action

**Preconditions:** Service running on declared port (`http://localhost:9475/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The `prod:admin` preset is seeded in `PresetRegistry` with `allowedTools: ["readStateTool", "adminActionTool"]`.

**Steps:**
1. Open `http://localhost:9475/` → App UI tab.
2. Set **Environment** to `prod` and **Role** to `admin`. The resolved-preset preview shows `allowedTools: readStateTool, adminActionTool`.
3. Type `Flush the CDN edge cache for region us-east-1` in the **Request** field.
4. Click **Submit**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `PRESET_RESOLVED` within 1 s. The right-pane detail shows the resolved preset (`presetId: prod:admin`, `modelId: claude-sonnet-4-6`, `allowedTools: [readStateTool, adminActionTool]`).
- Within 30 s the card reaches `COMPLETED`. The agent answer describes the cache flush outcome. The tool call log shows one `adminActionTool` entry with `status: INVOKED`.
- The audit summary shows `toolCallCount: 1`, `blockedToolCallCount: 0`.

## J2 — prod + guest blocked from privileged action

**Preconditions:** Service running. The `prod:guest` preset is seeded with `allowedTools: ["readStateTool"]` (no `adminActionTool`).

**Steps:**
1. Set **Environment** to `prod` and **Role** to `guest`. The resolved-preset preview shows `allowedTools: readStateTool`.
2. Type `Flush the CDN edge cache for region us-east-1` in the **Request** field.
3. Click **Submit**.

**Expected:**
- The card reaches `COMPLETED` (the task does not fail — the agent explains the constraint and returns a result).
- The agent answer states that cache flush requires admin access and the caller's current preset is `prod:guest`.
- The tool call log shows one `adminActionTool` entry with `status: BLOCKED`. The `outputSummary` names the tool and the active role.
- The audit summary shows `toolCallCount: 1`, `blockedToolCallCount: 1`.
- `adminActionTool` never executed — there is no side-effect on any simulated backend state.

## J3 — CI gate rejects a malformed preset file

**Preconditions:** Developer working on a new preset file. The Maven build is available.

**Steps:**
1. Add a file `src/main/resources/presets/test-broken.json` with content `{ "presetId": "prod:operator", "environment": "production", "role": "operator" }` (missing `modelId`, `allowedTools`, `instructionAddendum`; `environment` is `"production"` not `"prod"`).
2. Run the Maven build (`/akka:build` or `mvn package`).

**Expected:**
- The `validate` lifecycle phase runs `tools/validate-presets.sh`.
- The script reports at least two violations: `environment` value `"production"` is not in `{dev, staging, prod}`, and required fields `modelId`, `allowedTools`, `instructionAddendum` are missing.
- The Maven build exits non-zero. No JAR is produced.
- Removing or fixing `test-broken.json` and re-running the build succeeds.

## J4 — dev + admin gets diagnostic tools; prod + admin does not

**Preconditions:** Service running with mock LLM or a live model. The `dev:admin` preset is seeded with `allowedTools: ["readStateTool", "adminActionTool", "diagnosticTool"]`. The `prod:admin` preset has `allowedTools: ["readStateTool", "adminActionTool"]`.

**Steps:**
1. Submit a request with `dev` + `admin`: `Run a slow-query diagnostic for the last 5 minutes`.
2. Note the tool call log.
3. Submit the same request with `prod` + `admin`.
4. Note the tool call log for the second card.

**Expected:**
- The `dev:admin` card's tool call log shows `diagnosticTool` with `status: INVOKED`.
- The `prod:admin` card's tool call log shows `diagnosticTool` with `status: BLOCKED`, because `diagnosticTool` is not in `prod:admin`'s `allowedTools`.
- The resolved-preset preview in the UI reflects the difference: `dev:admin` lists three tools; `prod:admin` lists two.

## J5 — Preset resolution failure transitions to FAILED

**Preconditions:** Service running. The `staging:operator` preset has NOT been seeded (there is no `staging:operator` entry in `PresetRegistry`).

**Steps:**
1. Submit a request by calling `POST /api/preset-requests` directly with `{ "environment": "staging", "role": "operator", "requestText": "Show health", "submittedBy": "tester" }`.

**Expected:**
- The card appears with status `SUBMITTED`.
- `resolvePresetStep` calls `PresetRegistry.getPreset("staging:operator")` and receives empty.
- The workflow transitions to `FAILED` after step timeout or explicit not-found handling.
- The entity's `RequestFailed` event carries a reason string naming the missing preset key.
- The card shows status `FAILED` with the reason visible in the right-pane detail.
- No agent call was made — `ContextPresetAgent` was never invoked.
