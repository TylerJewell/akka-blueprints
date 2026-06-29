# Akka Sample: Concurrent Research Writer

A writing coordinator delegates topic research to two specialist agents running **in parallel**, then merges their outputs into one polished written report. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound request queue and the research tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.research-intel.concurrent-research-writer  ~/my-projects/concurrent-research-writer
cd ~/my-projects/concurrent-research-writer
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReportCoordinator** — AutonomousAgent that decomposes a writing topic into parallel work assignments and merges the workers' outputs into one final report.
- **SourceResearcher** — AutonomousAgent that gathers factual source material for a given subtopic.
- **DraftWriter** — AutonomousAgent that drafts a narrative section from a given writing brief.
- **ReportWorkflow** — Workflow that fans work out to SourceResearcher and DraftWriter in parallel, then calls ReportCoordinator for final assembly.
- **ReportEntity** — EventSourcedEntity holding the full report lifecycle.
- **ReportView** — projection the UI streams via SSE.
- **ReportEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the writing topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Report` record fields (e.g., add `wordCount` or `targetAudience`).
- `prompts/source-researcher.md` — narrow the researcher to a single subject area.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real search API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a writing topic → report enters `PLANNING`, then `IN_PROGRESS`, then `ASSEMBLED`.
2. Workers fail-fast → if either SourceResearcher or DraftWriter times out, the report enters `DEGRADED` with whichever partial output exists.
3. An output guardrail vets the assembled report; fabricated citations cause it to enter `BLOCKED`.
4. Wait after a successful assembly; the report's row in the UI shows an eval score.

## License

Apache 2.0.
