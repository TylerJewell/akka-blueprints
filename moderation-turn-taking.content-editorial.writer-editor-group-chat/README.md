# Akka Sample: Writer-Editor Group Chat

Type a topic. A Writer agent and an Editor agent take turns in a moderated group chat until the Editor approves the draft.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Integration: Runs out of the box. No host software required.
- One model-provider key option (asked for during generation if none is set):
  - `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` already exported, or
  - a mock model provider (no key), an env-file path, a secrets-store URI, or a type-once-in-session value.
  The key value is never written to disk.

## Generate the system

```sh
cp -r ./moderation-turn-taking.content-editorial.writer-editor-group-chat  ~/my-projects/writer-editor-group-chat
cd ~/my-projects/writer-editor-group-chat
```

(Optional) Edit `SPEC.md` — system name, model provider, round limit.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 chains `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` automatically and prints the listening URL when the service is up.

## What you'll get

- `WriterAgent` and `EditorAgent` (AutonomousAgent) — draft and critique.
- `GroupChatWorkflow` (Workflow) — the group chat manager running RequestToSpeak round-robin moderation.
- `ConversationEntity` (EventSourcedEntity) — the chat session and its turns.
- `TurnView` (View) — read model the UI streams.
- `TurnEvalConsumer` (Consumer) — event-triggered eval on each Editor decision.
- `TopicSimulator` and `StallMonitor` (TimedAction) — background topic feed and stall escalation.
- `ChatEndpoint` and `AppEndpoint` (HttpEndpoint) — the `/api` surface and the UI.

## Customise before generating

- System name and round limit — SPEC.md Sections 1 and 11.
- Model provider — SPEC.md Section 11 generation workflow.
- Writer and Editor behavior — `prompts/writer-agent.md`, `prompts/editor-agent.md`.

## What gets validated

- Submit a topic; a Writer turn then an Editor turn appear in the live transcript.
- The round-robin continues until the Editor approves or the round limit is reached.
- A blocked low-quality turn is recorded and re-requested.
- A stalled conversation escalates automatically.

See `reference/user-journeys.md` for the full acceptance journeys.

## License

Apache 2.0.
