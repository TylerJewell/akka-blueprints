# Akka Sample: Academic Research Team

A literature coordinator delegates publication discovery to a Paper Scout and trend interpretation to a Domain Analyst running **in parallel**, then synthesises their outputs into one academic research digest. Demonstrates the **delegation-supervisor-workers** coordination pattern with an embedded citation guardrail.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound query stream and the publication-discovery tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.research-intel.academic-research-team  ~/my-projects/academic-research-team
cd ~/my-projects/academic-research-team
```

(Optional) Edit `SPEC.md` to change the research domains the simulator drips, or remove the simulator entirely.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **LiteratureCoordinator** — AutonomousAgent that decomposes a research query into a scouting directive and an interpretation question, then synthesises results into one digest.
- **PaperScout** — AutonomousAgent that discovers relevant publications (modelled with seeded tools).
- **DomainAnalyst** — AutonomousAgent that identifies emerging trends and research gaps.
- **ResearchWorkflow** — Workflow that fans the work out to PaperScout and DomainAnalyst in parallel, then calls LiteratureCoordinator for synthesis.
- **ResearchDigestEntity** — EventSourcedEntity holding the digest's full lifecycle.
- **DigestView** — projection the UI streams via SSE.
- **DigestEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the research domains the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `ResearchDigest` record fields (e.g., add `noveltyScore`).
- `prompts/paper-scout.md` — narrow the agent to a single discipline (e.g., genomics only).
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real publication database API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a research query → digest enters `QUEUED`, then `SCANNING`, then `SYNTHESISED`.
2. Workers fail-fast → if either PaperScout or DomainAnalyst times out, the digest enters `DEGRADED` with whichever partial output exists.
3. The citation guardrail blocks a digest containing fabricated DOIs.
4. Wait after a successful synthesis; the digest row shows an eval score from `EvalSampler`.

## License

Apache 2.0.
