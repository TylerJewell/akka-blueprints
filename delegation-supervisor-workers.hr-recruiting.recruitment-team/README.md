# Akka Sample: Recruitment Team

A supervisor agent delegates resume matching and screening to worker agents, redacts protected attributes first, evaluates every screen decision, and pauses for a human before any candidate is advanced or rejected.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- An AI key — pick one path when `/akka:specify` asks (Lesson 25): a mock provider (no key), an existing env var (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`), an env file path, a secrets-store URI, or type a key once for this session.
- Host software: None. This blueprint runs out of the box — every external system is modeled inside the same service.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.hr-recruiting.recruitment-team ~/my-projects/recruitment-team
cd ~/my-projects/recruitment-team
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or the matching/screening criteria.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, then print the listening URL.

## What you'll get

- Three `AutonomousAgent`s: `MatchAgent` (candidate-to-role fit), `ScreenAgent` (advance/hold/reject screen), `SupervisorAgent` (aggregates worker output into one recommendation).
- One `Workflow` `ScreeningWorkflow` delegating sanitize → match → screen → supervise → await-decision → finalize.
- Two `EventSourcedEntity`s: `CandidateEntity` and `InboundApplicationQueue`.
- One `View` `CandidatesView` projecting candidates for the UI (queried, streamed via SSE).
- One `Consumer` `ApplicationConsumer` starting a workflow per inbound application.
- Three `TimedAction`s: `ApplicationSimulator`, `FairnessDriftMonitor`, `StuckDecisionMonitor`.
- Two `HttpEndpoint`s: `ScreeningEndpoint` (`/api`) and `AppEndpoint` (static UI).
- A single self-contained UI at `src/main/resources/static-resources/index.html` with five tabs.

## Customise before generating

- `SPEC.md` Section 1 — system name and one-line pitch.
- `SPEC.md` Section 11 — model provider and port if 9654 collides with another local service.
- `prompts/match-agent.md` and `prompts/screen-agent.md` — matching and screening criteria for your roles.

## What gets validated

The journeys in [`reference/user-journeys.md`](reference/user-journeys.md):

- An application is sanitized, matched, screened, and a recommendation appears in `AWAITING_DECISION`.
- A human approves a candidate and the record moves to `APPROVED`.
- A human rejects a candidate with a reason and the record moves to `REJECTED`.
- An untouched decision escalates after the configured wait.
- The fairness/drift watch flags a skew across recent decisions.

## License

Apache 2.0.
