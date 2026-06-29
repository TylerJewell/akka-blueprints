# Akka Sample: Discord AI-Powered Bot

A `ClassifierAgent` reads an inbound Discord message and routes it to a `CommunitySpecialist` or `TechnicalSpecialist` that owns the reply end-to-end. Demonstrates the **handoff-routing** coordination pattern layered with two governance mechanisms: a PII sanitizer that scrubs user messages before any model call, and a before-agent-response guardrail that checks every draft reply against a community-policy rubric.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound Discord message stream and the outbound reply surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.cx-support.discord-router  ~/my-projects/discord-router
cd ~/my-projects/discord-router
```

(Optional) Edit `SPEC.md` to point `MessageSimulator` at a real Discord webhook source, or to add additional routing categories (`MODERATION`, `ANNOUNCEMENTS`, etc.) and matching specialist agents.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MessageSimulator** — TimedAction firing every 30 s that drips canned Discord messages from a JSONL file into `MessageQueue`.
- **MessageQueue** — EventSourcedEntity append-only log of every inbound message (audit before redaction).
- **PiiSanitizer** — Consumer that redacts emails, usernames, phone numbers, and user ids before any LLM call.
- **ClassifierAgent** — typed Agent that routes the sanitized message to `COMMUNITY`, `TECHNICAL`, or `UNCLEAR`.
- **CommunitySpecialist** — AutonomousAgent that owns the `REPLY` task for community messages (questions, onboarding, general discussion).
- **TechnicalSpecialist** — AutonomousAgent that owns the `REPLY` task for technical messages (errors, SDK questions, integration help).
- **RoutingJudge** — typed Agent used by `RoutingEvalScorer` to grade every routing decision against a 1–5 rubric.
- **ReplyGuardrail** — typed Agent used by `DiscordWorkflow` to check every draft reply before it is published.
- **DiscordWorkflow** — Workflow per message: sanitize-handoff → classify → route → reply → guardrail check → publish.
- **MessageEntity** — EventSourcedEntity holding each message's lifecycle.
- **MessageView + DiscordEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **RoutingEvalScorer** — Consumer that listens for `MessageRouted` events and writes an inline eval score.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the routing taxonomy (add `MODERATION`, `ANNOUNCEMENTS`) and add matching specialist agents.
- `SPEC.md §5` — extend the `Message` record with deployer-specific fields (`guildId`, `channelId`, `userRole`, `threadId`).
- `prompts/classifier-agent.md` — adjust the routing thresholds (e.g., narrow what counts as `TECHNICAL` in your server's context).
- `prompts/community-specialist.md` / `prompts/technical-specialist.md` — encode your server's tone guidelines, escalation triggers, and banned-topic rules.
- `eval-matrix.yaml` — swap the in-process PII regex for an external redactor if your server surfaces usernames that require a more precise approach.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a community-flavoured message → it is sanitized, classified as `COMMUNITY`, handed off to `CommunitySpecialist`, and a reply is published.
2. Simulator drips a technical message → it is sanitized, classified as `TECHNICAL`, handed off to `TechnicalSpecialist`, and a reply is published.
3. An ambiguous message classifies as `UNCLEAR` and the workflow terminates in `ESCALATED` without any specialist being invoked.
4. A specialist draft that violates the before-agent-response guardrail (e.g. links to an unapproved external URL, echoes a `[REDACTED]` token) is blocked; the message lands in `BLOCKED` for moderator review.
5. The routing score (1–5) and rationale appear on every classified message within a few seconds of the routing decision.

## License

Apache 2.0.
