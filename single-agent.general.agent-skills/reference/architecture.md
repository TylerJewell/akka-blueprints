# Architecture: Modular Agent Skills Loader

This document narrates the four diagrams in [PLAN.md](../PLAN.md).

---

## Component Graph (Diagram 1)

The system has five Akka components arranged in four layers.

**Endpoint Layer** — `AgentOrchestrationEndpoint` is the sole entry point for all HTTP traffic. It accepts task submissions (`POST /tasks`), skill management operations (`POST /skills`, `PUT /skills/{id}/enable`, `PUT /skills/{id}/disable`), and query requests. It also holds the SSE connection for `GET /tasks`, pushing a `task-update` event each time a task transitions state.

**Agent Layer** — `TaskDispatchAgent` is the AI agent responsible for skill selection and invocation. It calls the `registry-lookup` tool (backed by `SkillRegistryEntity`) to discover which skills are enabled for a given capability, picks the best candidate, and then calls the skill tool. Inline in the Agent Layer is the **Guardrail**, a before-tool-invocation hook that validates the selected skill before any tool call reaches execution. The guardrail is not a separate component — it is a hook declared in the agent's configuration and wired to `SkillLoaderWorkflow`.

**Workflow Layer** — `SkillLoaderWorkflow` runs the four-step validation lifecycle (validate, load, verify, emit) for each task that passes the initial skill-selection phase. Because it is a Workflow, each step is durable: if the process restarts between steps, the workflow resumes from the last successfully completed step. Failure at any step causes the workflow to emit `SkillRejected` and mark the task `REJECTED`.

**Entity Layer** — `SkillRegistryEntity` is a Key-Value Entity keyed by `skillId`. It is the authoritative record of each registered skill's state. Both the guardrail (via `SkillLoaderWorkflow`) and the agent's `registry-lookup` tool read from this entity. Write operations (register, enable, disable) come only via `AgentOrchestrationEndpoint`.

**View Layer** — `SkillExecutionView` is a read model built from the event stream. It maintains one `TaskRecord` row per task, updated by `SkillLoaded`, `SkillRejected`, and task-progress events. The endpoint queries this view for `GET /tasks` and `GET /tasks/{taskId}`. The view is never directly written by business logic; it is always rebuilt from events.

---

## Request Sequence (Diagram 2)

The sequence diagram traces the full lifecycle of a task submission:

1. The client sends `POST /tasks` with a description and required capability.
2. `AgentOrchestrationEndpoint` routes to `TaskDispatchAgent`.
3. The agent calls `registry-lookup` on `SkillRegistryEntity` and receives a list of enabled skills.
4. The agent selects the best skill and issues a tool-invocation call.
5. The **guardrail** intercepts the call, reads the skill's `SkillDescriptor` from `SkillRegistryEntity`, and checks enabled status and capability membership.
6. **Happy path**: the guardrail approves → `SkillLoaderWorkflow` runs its three verification steps → emits `SkillLoaded` → agent executes the skill tool → task completes → endpoint returns 200.
7. **Rejected path**: the guardrail blocks → `SkillRejected` is emitted → task moves to `REJECTED` → endpoint returns 422 with reason.

Throughout the sequence, `AgentOrchestrationEndpoint` pushes SSE `task-update` events to any subscribed client each time the status changes.

---

## Task State Machine (Diagram 3)

Tasks move through six states:

- **PENDING** — the starting state after `POST /tasks` is accepted.
- **SKILL_SELECTED** — the agent has identified a candidate skill. This is the last state before the guardrail fires.
- **GUARDRAIL_PASSED** — the capability was verified in the registry and the skill is enabled. The workflow emits `SkillLoaded` at this transition.
- **EXECUTING** — the skill tool is actively running.
- **COMPLETED** — the skill returned a result. Terminal.
- **REJECTED** — the guardrail blocked the invocation, no matching skill was found, or the skill tool errored. Terminal.

Two rejection paths exist: from `PENDING` (no skill matches the capability) and from `SKILL_SELECTED` (guardrail blocks after selection). Both paths emit `SkillRejected` and return HTTP 422.

---

## Entity-Relationship (Diagram 4)

The ER diagram shows the data relationships:

- `SkillDescriptor` is the registry record. It has a one-to-many relationship with both `SkillLoaded` and `SkillRejected` events — one skill can be loaded or rejected across many tasks.
- `SkillLoadRequest` is the command that initiates the workflow. It maps one-to-one with a `TaskRecord` row.
- `TaskRecord` is the view row. It references a `skillId` (nullable until `SKILL_SELECTED`) and is updated by both `SkillLoaded` and `SkillRejected` events.

The separation between `SkillDescriptor` (entity state, owned by `SkillRegistryEntity`) and `TaskRecord` (view row, owned by `SkillExecutionView`) enforces the Akka principle that entities own state and views own derived read models.
