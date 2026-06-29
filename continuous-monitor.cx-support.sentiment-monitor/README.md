# Akka Sample: Sentiment Monitor

A continuous background worker polls Linear issue comments, runs sentiment scoring on each new comment, detects negative-trend patterns across issue threads, and dispatches alerts to configured Slack channels when a threshold is crossed. Demonstrates the **continuous-monitor** coordination pattern wired with two governance mechanisms: a periodic eval for drift detection on the sentiment classifier and a before-tool-call guardrail validating Slack channel scope before every alert dispatch.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A valid credential for the integrated service (Linear API token + Slack bot token) — sourced via the same five options as the LLM key. Akka records only the reference (env-var name, file path, secrets URI); never writes the value to disk.

## Generate the system

```sh
cp -r ./continuous-monitor.cx-support.sentiment-monitor  ~/my-projects/sentiment-monitor
cd ~/my-projects/sentiment-monitor
```

(Optional) Edit `SPEC.md` to point the `CommentPoller` at a real Linear workspace or to keep the in-memory simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CommentPoller** — TimedAction firing every 20 s that pulls new comments from a simulated Linear issue feed.
- **SentimentScoringAgent** — Agent (typed) that scores each comment on a −5 to +5 sentiment scale.
- **TrendAnalysisAgent** — AutonomousAgent that evaluates a window of scored comments and decides whether an alert should be dispatched.
- **AlertDispatchWorkflow** — Workflow that validates channel scope (via the before-tool-call guardrail) then calls the Slack alert tool.
- **IssueThreadEntity** — EventSourcedEntity holding each Linear issue thread's comment history and trend state.
- **SentimentEvalRunner** — TimedAction running every 60 minutes; samples recent scores against ground-truth labels to detect classifier drift.
- **ThreadView + SentimentEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated Linear feed for a real Linear webhook or polling integration.
- `SPEC.md §5` — extend the `IssueComment` record with org-specific fields (`priority`, `customerTier`, etc.).
- `prompts/sentiment-scorer.md` — adjust the sentiment rubric for your product domain.
- `eval-matrix.yaml` — add regulation anchors when deploying in a regulated sector.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated comment arrives → it is scored → the thread trend is updated.
2. When consecutive negative scores exceed the threshold → an alert is dispatched to Slack (simulated).
3. The before-tool-call guardrail blocks dispatch to channels not in the allowlist.
4. The eval runner detects and surfaces a drift signal when mock scores diverge from ground truth.

## License

Apache 2.0.
