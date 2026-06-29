# Akka Sample: Slack Assistant

A Slack-bot agent that monitors channels for questions and commands, routes each request to the appropriate workflow, and posts a structured response back to the thread. The agent receives message context as a task attachment — channel history is never inlined into the prompt.

Demonstrates the **single-agent** coordination pattern in the ops-automation domain. One `ChannelAssistantAgent` (AutonomousAgent) drives every response; the surrounding components prepare its input, guard its output, and audit the conversation. Two governance mechanisms are wired around the agent:

- A **secret sanitizer** that runs inside a Consumer before message context ever reaches the agent, stripping tokens, credentials, and Slack webhook URLs from channel history.
- A **before-agent-response guardrail** that validates every response before it is posted to the channel: well-formed reply JSON, allowed action types, no secret-looking strings in the output text.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A valid Slack credential for the integrated service. Obtain a Slack Bot Token (`xoxb-...`) with `channels:history`, `channels:read`, `chat:write`, and `chat:write.public` scopes from your Slack workspace settings. Supply this token using the same key-sourcing options listed above — Akka records only the reference, never the token value.

## Generate the system

```sh
cp -r ./single-agent.ops-automation.slack-assistant  ~/my-projects/slack-assistant
cd ~/my-projects/slack-assistant
```

(Optional) Edit `SPEC.md §3` to add channel-specific routing rules (e.g., restrict incident commands to an `#incidents` channel, add a custom trigger keyword list).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ChannelAssistantAgent** — an AutonomousAgent that receives a sanitized message context attachment and returns a typed `AssistantReply` with a response action and text.
- **MessageWorkflow** — orchestrates sanitize-wait → reply → audit per inbound Slack message.
- **MessageEntity** — an EventSourcedEntity holding the per-message lifecycle.
- **SecretSanitizer** — a Consumer that subscribes to `MessageReceived` events, scrubs secrets from the context window, and emits `MessageSanitized` back to the entity.
- **MessageView + MessageEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — add your workspace's Slack channel IDs and trigger keywords to the routing table.
- `SPEC.md §5` — extend `AssistantReply` with ops-specific fields (e.g., `runbookUrl`, `oncallHandle`, `incidentSeverity`).
- `prompts/channel-assistant.md` — narrow the agent's role (a SRE team would constrain it to incident triage and escalation; a DevOps team to deployment status queries).
- `eval-matrix.yaml` — wire a real secret-detection library (e.g., a Gitleaks-style regex bundle) under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A Slack message arrives → it is sanitized → the agent replies → the reply appears in the live list and the thread.
2. The agent returns a reply containing a secret-looking token on first try → the `before-agent-response` guardrail rejects it → the agent retries → a clean reply is posted.
3. A message context window containing `xoxb-` and `ghp_` tokens is processed — neither token appears in the agent task attachment or the posted reply.
4. Every recorded reply is visible in the App UI with its sanitizer report and guardrail decision count.

## License

Apache 2.0.
