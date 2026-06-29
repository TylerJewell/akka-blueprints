# Akka Sample: NexShift Agent

A single workforce-scheduling agent reads open shifts and employee availability, then proposes an optimized shift assignment. Schedule writes are guarded by a `before-tool-call` guardrail that validates every proposed assignment before it is committed — preventing double-bookings, constraint violations, and assignments that exceed regulatory hour limits.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a `before-tool-call` guardrail that intercepts every schedule mutation the agent attempts.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — employee and shift data live in-process and schedule mutations are handled by the entity layer.

## Generate the system

```sh
cp -r ./single-agent.hr-recruiting.shift-scheduler  ~/my-projects/nexshift
cd ~/my-projects/nexshift
```

(Optional) Edit `SPEC.md` to adjust the seeded shift dataset (swap the seed roles from `Nurse / Technician / Coordinator` to your own workforce categories) or to tighten the hour-limit rules applied by the guardrail.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **NexShiftAgent** — an AutonomousAgent that accepts a scheduling request (open shifts + employee roster) and emits a sequence of `AssignShift` tool calls to fill the schedule.
- **ScheduleWorkflow** — orchestrates build-roster → schedule → confirm per submitted scheduling run.
- **ScheduleEntity** — an EventSourcedEntity holding the per-run lifecycle.
- **AssignmentGuardrail** — intercepts every `AssignShift` tool call before it executes; rejects calls that violate double-booking, hour-limit, or qualification rules.
- **ScheduleView + ScheduleEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded shift dataset for your own (the JSONL file under `src/main/resources/sample-events/shifts.jsonl` after generation).
- `SPEC.md §5` — extend `ShiftAssignment` with site-specific fields (e.g., `costCenter`, `location`, `unionCode`).
- `prompts/nexshift-agent.md` — narrow the agent's role (a healthcare deployer would constrain it to licensed-care-staff placements; a retail deployer to cashier-vs-floor splits by predicted footfall).
- `eval-matrix.yaml` — point the guardrail's hour-limit rule at your jurisdiction's statutory cap (e.g., 48 h/week under the EU Working Time Directive).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A manager submits an open-shift list; within 30 s the schedule appears in the UI with every shift filled and no constraint violations.
2. The agent attempts an assignment that would double-book an employee; the `before-tool-call` guardrail rejects it; the agent reassigns to an available employee; the UI never shows the rejected assignment.
3. An employee whose weekly hours are already at the statutory cap is never assigned an additional shift.
4. The manager can view the full assignment list with each employee's total-hours count and the guardrail audit log showing which calls were blocked.

## License

Apache 2.0.
