# Akka Sample: Open Deep Research

A manager agent receives a research question, delegates web-browsing to a `WebBrowsingAgent` and document reading to a `FileReaderAgent`, then synthesises their retrieved content into a cited answer. Demonstrates the **delegation-supervisor-workers** coordination pattern with URL allow-listing guardrails, PII sanitization, per-step quality evaluation, and deployer-runtime monitoring.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the browser and file-reader tools are modelled inside the same service. No external browser binary or search API is required.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.research-intel.open-deep-research  ~/my-projects/open-deep-research
cd ~/my-projects/open-deep-research
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, URL allow-list, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ResearchManager** — AutonomousAgent that decomposes a question into a retrieval plan and synthesises the workers' retrieved content into a cited answer.
- **WebBrowsingAgent** — AutonomousAgent that fetches and extracts text from allowed URLs; pre-call guardrail enforces the URL allow-list.
- **FileReaderAgent** — AutonomousAgent that reads structured documents and returns parsed sections.
- **ResearchWorkflow** — Workflow that fans retrieval out to both workers in parallel, sanitizes PII from their outputs, then calls the manager for synthesis.
- **ResearchRunEntity** — EventSourcedEntity holding the run's lifecycle (queued → in-progress → answered / degraded / blocked).
- **ResearchView** — projection the UI streams via SSE.
- **ResearchEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the question simulator cadence or replace the simulator with a real inbound queue.
- `SPEC.md §5` — adjust the `ResearchRun` record fields (e.g., add a `confidenceScore`).
- `prompts/manager.md` — narrow the synthesis prompt to a specific subject domain.
- `eval-matrix.yaml` — tune the `before-tool-call` guardrail allow-list or add a secondary sanitization pass.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a question → run enters `QUEUED`, then `IN_PROGRESS`, then `ANSWERED`.
2. Worker timeout degrades the run — whichever partial retrieval exists is passed to the manager; the run enters `DEGRADED`.
3. The guardrail blocks a disallowed URL before the tool call executes; the run records the blocked URL.
4. The eval sampler scores a completed answer and the score appears in the App UI.
5. The human-oversight flag (deployer-runtime-monitoring) is queryable via the monitoring endpoint.

## License

Apache 2.0.
