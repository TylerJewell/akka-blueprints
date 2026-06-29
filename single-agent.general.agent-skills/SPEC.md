# Feature Specification: Modular Agent Skills Loader

## 1. Overview

This system provides a runtime mechanism for selecting and invoking agent skills on demand. Skills are self-contained capability units — folders containing a prompt, tool definitions, and metadata — stored in a registry. When a task arrives, the AI agent queries the registry to find a skill whose declared capabilities match the task requirement, then invokes that skill's tools.

Without runtime selection, skill bindings are hardcoded at deployment time. This makes it expensive to add new skills, disable faulty ones, or vary skill availability per tenant. The registry solves this: skills are first-class registry entries that can be enabled, disabled, or versioned without redeploying the agent.

The guardrail enforces that no tool is invoked unless the corresponding capability exists in the registry and the skill is currently enabled. This prevents stale bindings, misconfigured skills, or unauthorized capability expansion from reaching execution.

---

## 2. Goals

1. Allow skills to be registered, enabled, disabled, and versioned at runtime without agent redeployment.
2. Give the AI agent a reliable mechanism to discover which skills are currently available for a given capability.
3. Block tool invocation when the selected skill is disabled or the requested capability is absent from the registry.
4. Persist a full audit trail of every skill selection, guardrail decision, and task outcome as durable events.
5. Expose task status and skill registry state over HTTP and SSE so callers can observe progress in real time.
6. Keep the agent's decision logic separated from skill implementation so skills can be swapped independently.

---

## 3. Non-goals

1. Skill authoring tools or a skill IDE — skills are pre-authored folders, not generated at runtime.
2. Multi-agent coordination — this sample covers a single agent dispatching one skill per task.
3. Dynamic skill discovery from external package repositories — the registry is populated via API, not auto-crawled.
4. Role-based access control on skill invocation — authorization is out of scope; the guardrail checks capability presence, not caller identity.
5. Streaming skill output to the caller mid-execution — the SSE stream delivers task-level status updates, not token-level output.

---

## 4. Components

### SkillRegistryEntity (Key-Value Entity)

Keyed by `skillId`. Stores the authoritative record of a registered skill: its name, version, declared capabilities, and enabled/disabled status. Handles commands to register, enable, disable, and retrieve skills. Emits `SkillRegistered`, `SkillEnabled`, and `SkillDisabled` events.

### SkillLoaderWorkflow (Workflow)

Orchestrates the four-step lifecycle for each task:
1. **Validate** — call `SkillRegistryEntity` to confirm the requested capability exists and the skill is enabled.
2. **Load** — resolve the skill folder path and confirm the skill's tool manifest is accessible.
3. **Verify** — re-check capability list against registry (guards against concurrent disable).
4. **Emit** — produce `SkillLoaded` on success or `SkillRejected` on any failure.

The Workflow is durable: if the process restarts mid-step, it resumes from the last completed step.

### TaskDispatchAgent (Agent)

The AI agent that drives task execution. Receives a task description, calls the registry-lookup tool to find matching skills, selects the best candidate (highest version wins on tie), and invokes the skill tool. The guardrail intercepts the tool call before execution. On guardrail pass, the agent runs the skill and captures its output. On guardrail block, the agent surfaces the rejection reason without retrying.

### SkillExecutionView (View)

A query view built over the event log. Maintains one `TaskRecord` row per task, updated as events arrive. Supports queries by status, skillId, capability, and time range. Used by `AgentOrchestrationEndpoint` to serve `GET /tasks` and `GET /tasks/{taskId}`.

### AgentOrchestrationEndpoint (HTTP Endpoint)

The REST and SSE surface. Routes task submission to `SkillLoaderWorkflow`, skill management to `SkillRegistryEntity`, and task queries to `SkillExecutionView`. Streams live task-status updates over SSE on `GET /tasks`.

---

## 5. Data model

### SkillDescriptor (entity state — SkillRegistryEntity)

| Field | Type | Nullable | Description |
|---|---|---|---|
| skillId | String | No | Unique identifier (e.g. `web-search-v2`) |
| name | String | No | Human-readable display name |
| version | String | No | Semver string (e.g. `2.1.0`) |
| capabilities | List\<String\> | No | Capability tags this skill provides (e.g. `["web-search","url-fetch"]`) |
| enabled | boolean | No | Whether the skill may be invoked |
| loadedAt | Instant | No | When the skill was last registered or re-registered |

### SkillLoadRequest (command — SkillLoaderWorkflow)

| Field | Type | Nullable | Description |
|---|---|---|---|
| taskId | String | No | ID of the task triggering the load |
| requestedCapability | String | No | Capability the task requires |
| requestedBy | String | No | Identifier of the caller (service or user ID) |

### SkillLoaded (event)

| Field | Type | Description |
|---|---|---|
| skillId | String | Skill that was approved and loaded |
| taskId | String | Task this load is attached to |
| loadedAt | Instant | Timestamp of the guardrail pass |
| capabilities | List\<String\> | Capability list at time of approval |

### SkillRejected (event)

| Field | Type | Description |
|---|---|---|
| skillId | String | Skill that was rejected (may be empty if no skill matched) |
| taskId | String | Task this rejection is attached to |
| reason | String | Human-readable rejection reason |
| rejectedAt | Instant | Timestamp of the rejection |

### TaskRecord (view row — SkillExecutionView)

| Field | Type | Nullable | Description |
|---|---|---|---|
| taskId | String | No | Task identifier |
| skillId | String | Yes | Skill selected (null until SKILL_SELECTED) |
| status | TaskStatus | No | Current lifecycle status |
| capability | String | No | Requested capability |
| dispatchedAt | Instant | No | When task was submitted |
| completedAt | Instant | Yes | When task reached COMPLETED or REJECTED |

### TaskStatus (enum)

| Value | Description |
|---|---|
| PENDING | Task received; skill selection not yet started |
| SKILL_SELECTED | Agent has identified a candidate skill |
| GUARDRAIL_PASSED | Capability verified in registry; tool invocation approved |
| EXECUTING | Skill tool is running |
| COMPLETED | Skill returned a result; task closed |
| REJECTED | Guardrail blocked the invocation or no matching skill found |

---

## 6. API contract

See [reference/api-contract.md](reference/api-contract.md) for full request/response schemas and SSE event format.

Summary:

| Method | Path | Description |
|---|---|---|
| POST | /tasks | Submit a task |
| GET | /tasks/{taskId} | Get task status |
| GET | /tasks | SSE stream of task updates |
| GET | /skills | List all registered skills |
| POST | /skills | Register a new skill |
| PUT | /skills/{skillId}/enable | Enable a skill |
| PUT | /skills/{skillId}/disable | Disable a skill |
| GET | /skills/{skillId} | Get skill details |

---

## 7. Guardrail

The guardrail is a **before-tool-invocation** hook wired into `TaskDispatchAgent`. It fires once per tool call, before any skill tool executes.

**Logic:**

1. Extract the `skillId` and `capability` from the pending tool call.
2. Call `SkillRegistryEntity.get(skillId)` — if not found, reject immediately.
3. Confirm `descriptor.enabled == true` — if false, emit `SkillRejected` with reason `"skill-disabled"`.
4. Confirm `capability` is present in `descriptor.capabilities` — if absent, emit `SkillRejected` with reason `"capability-not-registered"`.
5. On all checks passing, allow the tool call to proceed and transition task to `GUARDRAIL_PASSED`.

The guardrail is **blocking**: a rejected check halts the task and returns HTTP 422 to the caller. The agent does not retry with the same skill on the same task.

---

## 8. AI agent behavior

`TaskDispatchAgent` operates in a single-turn loop per task:

1. **Receive** the task description from `AgentOrchestrationEndpoint`.
2. **Parse** the description to identify the required capability (e.g. `"web-search"`, `"image-classify"`).
3. **Call** the `registry-lookup` tool to retrieve all enabled skills that declare the required capability.
4. **Select** the best match: if multiple skills qualify, pick the one with the highest semver version.
5. **Invoke** the selected skill's primary tool. The guardrail intercepts this call before execution.
6. **On pass**: run the tool, capture the output, update task to `COMPLETED`, return result to endpoint.
7. **On block**: read the rejection reason, update task to `REJECTED`, return 422 with the reason.

The agent does not make additional tool calls after a guardrail rejection on the same task. If no skill matches the capability in step 4, the agent immediately emits `SkillRejected` with reason `"no-matching-skill"` without attempting any tool call.

---

## 9. Observability

- **SSE task stream** (`GET /tasks`): every task-status transition emits a `task-update` event with taskId, new status, skillId, and timestamp. Callers can subscribe and display live progress.
- **View queries**: `SkillExecutionView` supports filtered queries (by status, skillId, capability, time window) for dashboards and audits.
- **Event log**: every `SkillLoaded`, `SkillRejected`, `SkillRegistered`, `SkillEnabled`, and `SkillDisabled` event is durably stored in the Akka event journal, queryable for post-hoc analysis.
- **Workflow step log**: `SkillLoaderWorkflow` records each step transition (validate, load, verify, emit) so partial failures can be diagnosed without replaying the full event log.

---

## 10. Testing

- **Guardrail unit tests**: verify that each rejection condition (skill not found, skill disabled, capability absent) produces the correct `SkillRejected` event and HTTP 422 response.
- **SkillRegistryEntity tests**: confirm enable/disable/register state transitions and that duplicate registration updates version correctly.
- **SkillLoaderWorkflow integration tests**: run the full four-step workflow against an in-process entity, including failure injection at each step.
- **TaskDispatchAgent tests**: mock the registry-lookup tool to simulate multi-skill selection, tie-breaking by version, and no-match scenarios.
- **End-to-end tests**: POST a task via `AgentOrchestrationEndpoint`, assert the SSE stream delivers transitions through PENDING → SKILL_SELECTED → GUARDRAIL_PASSED → EXECUTING → COMPLETED for the happy path, and PENDING → REJECTED for the guardrail-block path.

---

## 11. Identity

```
folder:    single-agent.general.agent-skills
group:     io.akka.samples
artifact:  single-agent-general-agent-skills
package:   io.akka.samples.modularagentskillsloader
akka:      3.6.0
port:      9958
```

---

## 12. Constraints

- **Lesson 1** — Each Akka component type (Entity, Workflow, Agent, View, Endpoint) is used according to its intended contract; no mixing of state ownership across types.
- **Lesson 4** — `SkillRegistryEntity` is the single source of truth for skill state; `SkillExecutionView` is a derived read model and never owns canonical state.
- **Lesson 6** — Commands to `SkillRegistryEntity` are idempotent; re-registering a skill with the same ID updates the existing record rather than creating a duplicate.
- **Lesson 7** — `SkillLoaderWorkflow` steps are individually retriable; a transient failure in the validate step does not corrupt the load or verify steps.
- **Lesson 8** — The guardrail hook is declared in the agent configuration, not inlined into business logic, so it applies uniformly to all tool calls without per-tool annotations.
- **Lesson 9** — `TaskDispatchAgent` reads skill state via a tool call at dispatch time, never from a cached snapshot, so it always sees the latest enabled/disabled status.
- **Lesson 10** — SSE events carry only stable identifiers (taskId, skillId, status); the subscriber fetches full detail via `GET /tasks/{taskId}` if needed.
- **Lesson 11** — `SkillRejected` and `SkillLoaded` events are the durable record of guardrail decisions; the endpoint response is derived from these events, not the other way around.
- **Lesson 12** — `SkillExecutionView` is rebuilt from the event log on startup; no view state is stored outside the event journal.
- **Lesson 13** — The agent's system prompt (in `prompts/task-dispatch-agent.md`) is the sole description of agent behavior; no business rules are embedded in endpoint or workflow code.
- **Lesson 23** — All user-facing status strings use the canonical `TaskStatus` enum values; no ad-hoc strings are produced by the endpoint.
- **Lesson 24** — Mermaid state diagrams in `PLAN.md` use CSS overrides for state labels so status names render legibly in both light and dark themes.
- **Lesson 25** — The eval-matrix `integration_tier` field uses full descriptive values (`runs-out-of-the-box`, etc.) rather than short codes.
- **Lesson 26** — The UI mockup in `reference/ui-mockup.md` uses `data-tab` attribute-based tab switching; no JavaScript class toggling is used.

---

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
