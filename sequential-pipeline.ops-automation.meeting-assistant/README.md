# Akka Sample: Meeting Assistant

Submit meeting notes. The system redacts personal data, extracts action items, then creates Trello cards and posts a Slack summary.

## Prerequisites

- Claude Code with the Akka plugin installed. See the Akka install docs.
- A model-provider API key, sourced one of five ways during generation (mock / env-var name / env-file path / secrets URI / type-once). No key is ever written to disk.
- Integration form: **Real service via env var.** The simulated Trello and Slack targets run in-process and need nothing. To write to real Trello and Slack, supply `TRELLO_API_KEY` (with `TRELLO_TOKEN`) and `SLACK_BOT_TOKEN` through the same key-sourcing options as the model key. Without those, the system uses the in-process simulators and still runs end to end.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` to change the system name, model provider, or the allowlisted Trello board and Slack channel.
3. In Claude Code, run:

```
/akka:specify @SPEC.md
```

That single command scaffolds the specification and then auto-chains `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build`. When the service is up it prints the listening URL.

## What you'll get

- `ActionExtractorAgent` — an AutonomousAgent that turns a redacted transcript into a summary plus a list of action items.
- `PipelineWorkflow` — a Workflow running `sanitizeStep` → `extractStep` → `dispatchStep`.
- `MeetingEntity` — an EventSourcedEntity holding each meeting's lifecycle.
- `InboundNotesQueue` — an EventSourcedEntity recording each submission.
- `MeetingsView` — the read model the UI lists and streams.
- `NotesConsumer` — starts a workflow per submission.
- `NotesSimulator` and `DispatchRetryMonitor` — TimedActions.
- `MeetingEndpoint`, `SimEndpoint`, `AppEndpoint` — the HTTP surface, the in-process Trello/Slack receivers, and the UI.

## Customise before generating

- System name and short name — `SPEC.md` Section 1.
- Model provider default — `SPEC.md` Section 11.
- Allowlisted Trello board id and Slack channel — `SPEC.md` Section 8 and `eval-matrix.yaml` control G1.
- Redaction rules for the PII sanitizer — `SPEC.md` Section 8 and `eval-matrix.yaml` control S1.

## What gets validated

The journeys in `reference/user-journeys.md`:

- Submit notes and watch action items get extracted.
- Personal data in the transcript is redacted before the agent sees it.
- Action items fan out to Trello cards and a Slack message.
- A blocked dispatch (non-allowlisted target) is refused by the guardrail.

## License

Apache 2.0.
