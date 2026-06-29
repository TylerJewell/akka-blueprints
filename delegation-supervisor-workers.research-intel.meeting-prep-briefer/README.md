# Akka Sample: Meeting Prep Briefer

Type a meeting topic and a list of participants. A supervisor agent delegates research to worker agents, then a briefing document comes back with background, talking points, and questions.

## Prerequisites

- Claude Code with the Akka plugin installed ([install docs](https://doc.akka.io/)).
- One model-provider API key, sourced any of five ways at generation time (env var already set, a named env var, an env file you maintain, a secrets-store URI, or typed once into the session). You can also pick a mock model that needs no key.
- Host software: None. This blueprint runs out of the box — every external research surface is modeled in-process with Akka components.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` — system name, model provider, the canned participants and topics.
3. In Claude Code, run:

```
/akka:specify @SPEC.md
```

That single command scaffolds the specification and, per `SPEC.md` Section 12, auto-chains `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build`. When the service is up, Claude prints the listening URL.

## What you'll get

- A `BriefingSupervisor` agent that turns a meeting request into a research plan.
- Worker agents: `ParticipantResearcher`, `TopicResearcher`, and a `BriefingComposer`.
- A `BriefingWorkflow` that fans research out to workers and gathers the results.
- A `BriefingEntity` event-sourced lifecycle, a `BriefingsView` read model, a request consumer, a request simulator, and the two HTTP endpoints (API + UI).
- A before-tool-call guardrail gating research lookups and a PII sanitizer redacting participant data before storage.
- The 5-tab UI shell (Overview / Architecture / Risk Survey / Eval Matrix / App UI).

## Customise before generating

- System name and port live in `SPEC.md` Section 1 and Section 11.
- Model provider defaults are in Section 11; the generator picks whichever key is present.
- The canned participants and meeting topics are in `reference/data-model.md` and the sample data referenced from Section 11.

## What gets validated

The acceptance journeys in `reference/user-journeys.md`:

1. Submit a meeting request via the API and watch the briefing reach `READY`.
2. The composed briefing carries background, talking points, and questions.
3. Participant data is PII-redacted in stored profiles.
4. The simulator seeds requests on its own every 30 seconds.

## License

Apache 2.0.
