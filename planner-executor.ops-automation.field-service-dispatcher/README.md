# Akka Sample: Scheduling Operations Agent

A DispatcherAgent plans a day's technician schedule, breaks it into assignment tasks, dispatches each to a RouteOptimizerAgent or AvailabilityAgent, records the outcome on a schedule ledger and a fairness ledger, and replans when an assignment fails or a technician becomes unavailable.

Demonstrates the **planner-executor** coordination pattern with embedded governance — pre-dispatch guardrails, periodic fairness monitoring, operator pause control, and output sanitization.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration tier (Runs out of the box): **None.** Technician rosters, work-order queues, and geographic routing are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.ops-automation.field-service-dispatcher  ~/my-projects/field-service-dispatcher
cd ~/my-projects/field-service-dispatcher
```

(Optional) Edit `SPEC.md` to change the scheduling window, technician roster size, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DispatcherAgent** — AutonomousAgent that maintains a schedule ledger (known work orders, unassigned slots, plan, current dispatch) and a fairness ledger (per-technician assignment counts, travel loads, last-assigned timestamps). Decides which specialist to call next. Replans when three consecutive assignments fail.
- **RouteOptimizerAgent** — AutonomousAgent that assigns a work order to the closest available technician from fixture data.
- **AvailabilityAgent** — AutonomousAgent that checks a technician's on-shift window and current workload from fixture data.
- **ScheduleWorkflow** — Workflow driving plan → check-halt → propose → guardrail → dispatch → sanitize → record → fairness-eval → decide loop, with replan and halt branches.
- **ScheduleEntity** — EventSourcedEntity holding the schedule lifecycle, both ledgers, and the final dispatch summary.
- **OperatorControlEntity** — EventSourcedEntity holding the operator pause flag.
- **WorkOrderQueue** — EventSourcedEntity audit log of incoming work orders.
- **ScheduleView** — projection used by the UI.
- **WorkOrderConsumer** — Consumer subscribing to `WorkOrderQueue` events; starts a `ScheduleWorkflow` per work order.
- **WorkOrderSimulator** — TimedAction dripping a sample work order every 90 seconds.
- **StalledScheduleMonitor** — TimedAction marking schedules stuck in `DISPATCHING` past 5 minutes as `STALLED`.
- **ScheduleEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the work-order prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Schedule` record fields (e.g., add `priorityScore`).
- `prompts/dispatcher.md` — narrow the dispatcher to a specific service type (e.g., HVAC-only).
- `eval-matrix.yaml` — add an `eval-event` control if you want per-assignment quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a work order → dispatcher plans, assigns technicians, completes within ~3 minutes.
2. Inject an out-of-policy dispatch (assign a technician outside their shift window) → guardrail blocks it; dispatcher replans; schedule either completes via a different assignment or fails gracefully.
3. Submit a work order and click **Pause new dispatches** while it is `DISPATCHING` → in-flight assignment finishes; no further dispatches occur; schedule moves to `PAUSED`.
4. A fixture technician record containing a credential-shaped string is scrubbed before it reaches the dispatcher's next prompt.
5. Periodic fairness monitoring detects a technician receiving significantly more assignments than peers → a `FairnessAlert` is recorded on the fairness ledger.

## License

Apache 2.0.
