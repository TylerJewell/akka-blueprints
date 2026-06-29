# Akka Sample: Reddit Search

A single browser-research agent accepts a search topic, opens Reddit pages via a headless browser, and returns a structured `ResearchReport` containing ranked post summaries, sentiment signals, and extracted themes. The agent is governed by a before-tool-call guardrail that validates every navigation URL before the browser acts on it, and a budget-based halt that terminates runaway sessions automatically.

Demonstrates the **single-agent** coordination pattern in the research-intel domain. One `BrowserResearchAgent` (AutonomousAgent) drives the entire extraction. The surrounding components only schedule its work and audit its output.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A headless-browser runtime. Pick one at scaffold time:
  - **Playwright** — installs locally via `playwright install chromium`; no external account required.
  - **E2B** — hosted sandbox API; set `E2B_API_KEY` in your environment.
  - **Modal** — serverless runtime; `modal token new` once per workstation.
  - **Robocorp** — RPA cloud; configure `RC_API_KEY` and workspace id.

  If none of the above is available, pick the **mock browser** option and the agent returns deterministic seeded results without any real HTTP traffic.

## Generate the system

```sh
cp -r ./single-agent.research-intel.browser-research-agent  ~/my-projects/reddit-search
cd ~/my-projects/reddit-search
```

(Optional) Edit `SPEC.md` to point at different seed subreddits, change the extraction depth, or swap the browser runtime choice in Section 11.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BrowserResearchAgent** — an AutonomousAgent that accepts a research topic, opens Reddit search and subreddit pages via its registered browser tool, and returns a typed `ResearchReport`.
- **ResearchWorkflow** — orchestrates enqueue → browse → extract → score steps for each submitted topic.
- **ResearchJobEntity** — an EventSourcedEntity holding per-job lifecycle.
- **NavigationGuardrail** — validates every URL before the browser tool executes; blocks off-domain and disallowed-path navigation.
- **BudgetEnforcer** — deterministic rule that terminates the agent session once page-visit budget is consumed.
- **ResearchView + ResearchEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded topic list or change the subreddit allow-list.
- `SPEC.md §5` — extend `ResearchReport` with domain-specific fields (e.g., `productMentions`, `sentimentBreakdown`, `topContributors`).
- `prompts/browser-research-agent.md` — narrow the extraction focus (a developer-tools company might restrict to technical threads; a brand team might target sentiment signals only).
- `eval-matrix.yaml` — wire a real URL allow-list (e.g., a corporate domain register) by naming it under the guardrail's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a research topic → the agent browses Reddit → a `ResearchReport` appears in the UI with ranked post summaries.
2. The agent attempts to navigate to an off-domain URL — the `before-tool-call` guardrail blocks it and the agent continues within Reddit.
3. A long-running search session consumes its page-visit budget — the halt fires, the job lands in `BUDGET_EXHAUSTED`, and the partial report is preserved.
4. A topic with no results produces a `ResearchReport` with an empty posts list and a clear `noResultsReason`.

## License

Apache 2.0.
