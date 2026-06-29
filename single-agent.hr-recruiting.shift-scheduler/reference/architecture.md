# Architecture — nexshift

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM that emits tool calls rather than returning a single verdict. `ScheduleEndpoint` accepts a scheduling request, writes `ScheduleRequested` onto `ScheduleEntity`, and starts a `ScheduleWorkflow` instance. The workflow's `buildRosterStep` reads the entity to confirm the request is persisted, then `scheduleStep` calls `NexShiftAgent` — the single AutonomousAgent — with the open shifts and employee roster as task instructions. The agent iterates through shifts, emitting `AssignShift` tool calls for each one. Every tool call passes through `AssignmentGuardrail` before it reaches the entity: valid calls are forwarded to `ScheduleEntity.recordAssignment`; rejected calls return a structured error to the agent loop so the agent can reassign to a different employee. Once the agent has processed all shifts, it returns a `ScheduleDraft` summarizing the run. The workflow writes `DraftRecorded`. The manager then issues a confirm command via `PUT /api/schedules/{id}/confirm`, which transitions the entity to `CONFIRMED`. `ScheduleView` projects every entity event into a read-model row; `ScheduleEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. `AssignmentGuardrail` is a deterministic constraint checker — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). The agent loop may call `AssignShift` many times before the task completes — once per open shift at minimum, more if some calls are rejected and retried. Each rejected call returns immediately as a tool result; the agent reads it and tries the next employee without a full round-trip to the model.

The manager confirm step is an explicit human gate between `DRAFT` and `CONFIRMED`. The confirm window is bounded by `confirmStep`'s 3600 s timeout; if the manager does not act, the run expires to `FAILED`.

## State machine

Six states. The paths:

- The happy path is `PENDING → BUILDING → SCHEDULING → DRAFT → CONFIRMED`.
- Three failure transitions land in `FAILED`: bad input during `PENDING`, agent timeout or budget exhaustion during `SCHEDULING`, and confirm-window expiry during `DRAFT`. A `FAILED` run's prior data is preserved on the entity — managers can inspect the partial draft and resubmit.
- There is no automatic promotion from `DRAFT` to `CONFIRMED`. The human confirmation step is always required. The blueprint stops at `CONFIRMED` — downstream payroll or HR-system integrations are deployer responsibility.

## Entity model

`ScheduleEntity` is the source of truth. It emits six event types. `ScheduleView` projects every event into a row used by the UI. `ScheduleWorkflow` both reads (`getSchedule`) and writes (`markBuilding`, `markScheduling`, `recordDraft`, `fail`) on the entity. `AssignmentGuardrail` writes individual `ShiftAssignment` records to the entity on each accepted tool call. The relationship between `NexShiftAgent` and `ScheduleDraft` is "returns" — the agent's task result is the draft summary.

## Guardrail flow

For any assignment that lands in the entity log, the tool call passed through `AssignmentGuardrail` and satisfied all four checks:

1. **Availability** — the employee's availability windows cover the shift's start and end times.
2. **No overlap** — no other assignment in this run occupies a conflicting window for the same employee.
3. **Hour cap** — adding the shift's duration does not push the employee over `weeklyHoursCap`.
4. **Qualification match** — the employee holds every qualification the shift requires.

Each check is independent. A call that fails check 3 (hour cap) is rejected with reason `hour-cap-exceeded` even if it would have passed checks 1, 2, and 4. The agent reads all four possible rejection reasons from the guardrail spec and knows which employees to avoid on retry.
