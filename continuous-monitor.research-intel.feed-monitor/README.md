# Akka Sample: Feed Monitor

A scheduled background worker polls RSS and web feeds, summarizes and classifies each item with an AI, and posts selected notifications to Slack. Demonstrates the **continuous-monitor** coordination pattern with two governance mechanisms: a before-tool-call guardrail on outbound Slack posts and a periodic eval that monitors summary quality and job liveness.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A valid Slack credential for posting to a channel. Supply it the same way as the model-provider key: existing env var (`SLACK_BOT_TOKEN`), env file, secrets-store URI, or typed once in the session. Akka never writes the token value to disk.

## Generate the system

```sh
cp -r ./continuous-monitor.research-intel.feed-monitor  ~/my-projects/feed-monitor
cd ~/my-projects/feed-monitor
```

(Optional) Edit `SPEC.md` to point `FeedPoller` at real RSS URLs, adjust the Slack channel targets, or swap the simulated feed source for a live HTTP fetch.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **FeedPoller** — TimedAction firing every 60 s that fetches simulated feed items from a JSONL fixture.
- **FeedItemEntity** — EventSourcedEntity tracking each feed item through its lifecycle (received → summarized → classified → notified / suppressed / failed).
- **SummaryAgent** — Agent (typed) that produces a concise summary plus extracted topics for each item.
- **ClassifierAgent** — Agent (typed) that scores relevance and decides whether to notify or suppress.
- **FeedWorkflow** — Workflow per feed item orchestrating: summarize → classify → (if notify) post to Slack.
- **SlackNotifier** — Consumer that formats and dispatches outbound Slack posts; the before-tool-call guardrail is wired here.
- **FeedView** — View projecting per-item state rows for the UI.
- **EvalRunner** — TimedAction firing every 30 minutes; samples recent items, scores summary quality and classification accuracy.
- **FeedEndpoint + AppEndpoint** — REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — replace the JSONL fixture with real RSS endpoints by switching `FeedPoller` to an HTTP fetch.
- `SPEC.md §5` — add `sourceTag`, `publishedAt`, or `authorityScore` fields to `FeedItem`.
- `prompts/summary-agent.md` — narrow the summary to a specific topic domain (e.g., "enterprise AI regulation").
- `eval-matrix.yaml` — wire the channel-scope guardrail to a real Slack channel allow-list instead of the in-process stub.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated feed item arrives → it is summarized → classified → posted to Slack (simulated) within 60 s.
2. An item classified as SUPPRESS never reaches the Slack notifier; the guardrail also blocks any attempt to post it.
3. The eval runner scores at least one posted item within 30 minutes and the score appears in the UI.
4. An item whose summary meets the channel-scope guardrail's allow-list passes; one that targets a blocked channel is rejected.

## License

Apache 2.0.
