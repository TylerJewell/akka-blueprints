# Akka Sample: Ambient Email Assistant

An AI agent reads incoming email, classifies it, drafts replies or schedules meetings, and pauses for a human to approve before any message is sent or calendar event is created.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Integration credential: a valid Gmail OAuth token or service-account credential for the connected mailbox. Source this credential the same way as the LLM key — env var, env file path, secrets-store URI, or type-once in the session. Never write the value to disk.

## Generate the system

```sh
cp -r ./human-in-loop-gate.ops-automation.ambient-email-assistant  ~/my-projects/ambient-email-assistant
cd ~/my-projects/ambient-email-assistant
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or mailbox polling interval.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `TriageAgent` (AutonomousAgent) — reads an email's subject and body, classifies it, and returns a typed `EmailClassification{category, urgency, suggestedAction}`.
- `ReplyDraftAgent` (AutonomousAgent) — composes a reply draft and returns a typed `ReplyDraft{subject, body}`.
- `MeetingSchedulerAgent` (AutonomousAgent) — proposes a calendar event and returns a typed `MeetingProposal{title, proposedTime, attendees}`.
- `EmailWorkflow` (Workflow) — a 4-task graph: triage → draft action → await human approval → execute action.
- `EmailThreadEntity` (EventSourcedEntity) — the email thread lifecycle and its events.
- `ThreadsView` (View) — a read model the UI queries and streams over SSE.
- `EmailEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/triage-agent.md` — classification categories and urgency rules.
- `prompts/reply-draft-agent.md` — tone and reply style.
- `prompts/meeting-scheduler-agent.md` — scheduling constraints.

## What gets validated

- Submitting an email thread triggers triage and the thread appears in `TRIAGED` with a classification.
- Approving a drafted reply transitions the thread to `REPLY_SENT`.
- Approving a meeting proposal transitions the thread to `MEETING_SCHEDULED`.
- Dismissing a thread moves it to terminal `DISMISSED`.
- The send-email and create-event tool calls never run without explicit approval.
- PII in email content is redacted before being logged outside the workflow boundary.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
