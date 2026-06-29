# Akka Sample: Meeting To Tasks

Reads meeting notes, extracts a task list, gates it on a human approval, then writes the tasks to a Trello board, exports a CSV, and posts a Slack notification.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- An AI model key — one of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`. If none is set, `/akka:specify` will offer a mock model and four other key-sourcing options; no key value is ever written to disk.
- Integration: **Real service via env var.** The outbound Trello and Slack writes run against canned in-process facades by default. To hit the real services, supply valid credentials (`TRELLO_API_KEY`, `TRELLO_TOKEN`, `SLACK_BOT_TOKEN`) through the same key-sourcing options as the model key. No host software beyond a valid credential is required.

## Generate the system

```sh
cp -r ./sequential-pipeline.ops-automation.meeting-to-tasks  ~/my-projects/meeting-to-tasks
cd ~/my-projects/meeting-to-tasks
```

(Optional) Edit `SPEC.md` — system name, model provider, the canned meeting notes, or the redaction rules.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. Section 12 of `SPEC.md` instructs Claude to continue through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` without stopping.

## What you'll get

- One request/response `Agent` (TaskExtractionAgent) that turns raw notes into a typed task list.
- One `Workflow` (MeetingPipelineWorkflow) running the ordered steps sanitize → extract → await-approval → push-to-board → export-csv → notify.
- One `EventSourcedEntity` (MeetingEntity) tracking each meeting's lifecycle.
- One `View` (MeetingsView) projecting meetings into a read model streamed over SSE.
- One `TimedAction` (NotesSimulator) that drips canned meeting notes so the pipeline runs without any UI interaction.
- Two `HttpEndpoint`s — the JSON/SSE API and the static App UI.
- In-process Trello and Slack facades that flip to the real services when their env vars are set.

## Customise before generating

- `SPEC.md` Section 1 — the system name and the App UI title.
- `SPEC.md` Section 11 — the model provider default and the Trello/Slack credential references.
- `prompts/task-extraction-agent.md` — how tasks are pulled out of notes.
- `reference/data-model.md` — the redaction rules applied by the PII sanitizer.

## What gets validated

The user journeys in `reference/user-journeys.md`:

- Submit meeting notes and watch a task list appear in `AWAITING_APPROVAL`.
- Approve the task list and watch the Trello push, CSV export, and Slack notification complete.
- Reject a task list and watch the pipeline stop at `REJECTED`.
- Confirm attendee emails and names are redacted before any external write.
- Watch the simulator seed a meeting with no UI interaction.

## License

Apache 2.0.
