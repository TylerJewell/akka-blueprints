# User Journeys: Modular Agent Skills Loader

---

## Journey 1: Happy path — task dispatched and completed

**Preconditions**
- Skill `web-search-v2` (version `2.1.0`, capabilities `["web-search","url-fetch"]`, enabled `true`) is registered in `SkillRegistryEntity`.
- The local runtime is running on port `9958`.

**Steps**
1. Client sends `POST /tasks` with `{"description":"Find today's top news headlines","capability":"web-search","requestedBy":"user-1"}`.
2. `AgentOrchestrationEndpoint` creates task `task-001` in `PENDING` state and returns `202 Accepted`.
3. `TaskDispatchAgent` calls the `registry-lookup` tool with `capability=web-search`. Registry returns `[web-search-v2]`.
4. Agent selects `web-search-v2` (only candidate). Task transitions to `SKILL_SELECTED`.
5. Agent invokes the `web-search-v2` primary tool. Guardrail intercepts — checks `SkillRegistryEntity`: skill is enabled, `web-search` is in capabilities. Guardrail passes.
6. `SkillLoaderWorkflow` runs validate → load → verify steps. Emits `SkillLoaded`. Task transitions to `GUARDRAIL_PASSED`.
7. Agent executes the skill tool. Task transitions to `EXECUTING`.
8. Skill returns result. Task transitions to `COMPLETED`. Endpoint returns result to client.
9. SSE stream delivers `task-update` events at each transition (PENDING → SKILL_SELECTED → GUARDRAIL_PASSED → EXECUTING → COMPLETED).

**Expected outcome**: `GET /tasks/task-001` returns `status: "COMPLETED"`, `skillId: "web-search-v2"`, non-null `completedAt`.

---

## Journey 2: Guardrail block — selected skill is disabled

**Preconditions**
- Skill `data-extract-v1` (capabilities `["data-extract","csv-parse"]`) is registered but `enabled: false` (previously disabled via `PUT /skills/data-extract-v1/disable`).

**Steps**
1. Client sends `POST /tasks` with `{"description":"Extract rows from uploaded CSV","capability":"data-extract"}`.
2. Endpoint creates task `task-002` in `PENDING` state.
3. `TaskDispatchAgent` calls `registry-lookup` with `capability=data-extract`. Registry returns `[data-extract-v1]` (disabled skills are included in the lookup result so the agent can attempt selection — the guardrail is the enforcement point).
4. Agent selects `data-extract-v1`. Task transitions to `SKILL_SELECTED`.
5. Agent invokes `data-extract-v1` tool. Guardrail checks `SkillRegistryEntity`: `enabled == false`. Guardrail blocks.
6. `SkillRejected` event emitted with `reason: "skill-disabled"`. Task transitions to `REJECTED`.
7. Endpoint returns `422 Unprocessable Entity` with `{"taskId":"task-002","status":"REJECTED","reason":"skill-disabled"}`.

**Expected outcome**: `GET /tasks/task-002` returns `status: "REJECTED"`. SSE stream delivers `task-update` event with `status: "REJECTED"`.

---

## Journey 3: Guardrail block — capability not in registry

**Preconditions**
- No skill in the registry declares the capability `"video-transcribe"`.

**Steps**
1. Client sends `POST /tasks` with `{"description":"Transcribe attached video file","capability":"video-transcribe"}`.
2. Endpoint creates task `task-003` in `PENDING` state.
3. `TaskDispatchAgent` calls `registry-lookup` with `capability=video-transcribe`. Registry returns an empty list.
4. Agent finds no matching skill. Without making any tool call, agent emits `SkillRejected` with `reason: "no-matching-skill"`. Task transitions to `REJECTED` directly from `PENDING`.
5. Endpoint returns `422` with `{"taskId":"task-003","status":"REJECTED","reason":"no-matching-skill"}`.

**Expected outcome**: `GET /tasks/task-003` returns `status: "REJECTED"`, `skillId: null`, `completedAt` non-null. No guardrail hook fires because no tool call was attempted.

---

## Journey 4: Skill registration and first use

**Preconditions**
- No skill with capability `"summarize"` exists in the registry.

**Steps**
1. Operator sends `POST /skills` with `{"skillId":"summarize-v1","name":"Text Summarizer","version":"1.0.0","capabilities":["summarize","text-extract"],"enabled":true}`.
2. Endpoint registers the skill in `SkillRegistryEntity`. Emits `SkillRegistered` event. Returns `201 Created` with the full `SkillDescriptor`.
3. Operator confirms via `GET /skills/summarize-v1` — `enabled: true`.
4. Client submits `POST /tasks` with `{"description":"Summarize the attached report","capability":"summarize"}`.
5. `TaskDispatchAgent` calls `registry-lookup` with `capability=summarize`. Returns `[summarize-v1]`.
6. Agent selects `summarize-v1`. Guardrail checks: enabled, capability present. Guardrail passes.
7. Workflow runs, emits `SkillLoaded`. Agent executes skill. Task completes.

**Expected outcome**: task completes successfully. `GET /skills/summarize-v1` remains `enabled: true`. `SkillExecutionView` shows one `COMPLETED` task with `skillId: "summarize-v1"`.
