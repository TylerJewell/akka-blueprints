# Akka Sample: Guided Intake Agent

A conversation-state-driven agent that follows a declarative goal-and-plan to elicit information from a user across multiple turns, replanning when answers are incomplete or off-topic. Demonstrates the **planner-executor** coordination pattern with a before-response guardrail and a PII sanitizer wired into every turn.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Intake goals and simulated user responses are seeded from fixture files included in the service.

## Generate the system

```sh
cp -r ./planner-executor.cx-support.guided-intake-agent  ~/my-projects/guided-intake-agent
cd ~/my-projects/guided-intake-agent
```

(Optional) Edit `SPEC.md` to change the intake goal definitions, add question topics, or adjust the turn budget.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **IntakeAgent** — AutonomousAgent that owns a conversation plan, decides which question to ask next, detects when a goal is met, and replans when a user's answer is incomplete or diverges from the topic.
- **SanitizerAgent** — AutonomousAgent that scrubs PII from each user turn before it lands on the conversation record.
- **ConversationWorkflow** — Workflow that drives the plan → ask → receive → sanitize → evaluate → replan loop, plus the goal-met and turn-budget-exceeded exit branches.
- **ConversationEntity** — EventSourcedEntity holding the full conversation state: goal, plan, turn log, and outcome.
- **GoalCatalogEntity** — EventSourcedEntity holding operator-defined intake goals and their question sets.
- **ConversationView** — projection used by the UI.
- **ConversationRequestConsumer** — Consumer that subscribes to the session queue and starts a workflow per new session.
- **SessionQueue** — EventSourcedEntity that is the audit log of submitted intake sessions.
- **SessionSimulator** — TimedAction that drips a sample intake session every 90 s.
- **StaleSessionMonitor** — TimedAction that marks sessions stuck in `ELICITING` past 10 minutes as `ABANDONED`.
- **ConversationEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded intake goals or the turn budget per goal.
- `SPEC.md §5` — adjust the `Conversation` record fields (e.g., add `confidenceScore`).
- `prompts/intake-agent.md` — narrow the agent to a specific industry vertical or question domain.
- `eval-matrix.yaml` — add an `eval-event` control if you want per-turn quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Start an intake session → agent asks questions turn-by-turn, detects goal met, produces a filled `IntakeSummary`.
2. A user turn containing a topic violation triggers the before-response guardrail; the agent re-asks within policy.
3. A user turn containing a name, email address, or phone number is scrubbed before the turn lands on the conversation record.
4. An operator halt drains gracefully: the in-flight turn completes, no new questions are sent, the session ends in `HALTED`.

## License

Apache 2.0.
