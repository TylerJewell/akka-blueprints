# Akka Sample: HITL Deep Research

A research query is decomposed by a planning agent, each sub-topic is investigated by a research agent, findings are synthesised, and the assembled report is paused at a human review gate before final delivery.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.research-intel.deep-research-hitl  ~/my-projects/deep-research-hitl
cd ~/my-projects/deep-research-hitl
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `PlannerAgent` (AutonomousAgent) — decomposes a research query into a set of sub-topics; returns a typed `ResearchPlan{queryId, subTopics}`.
- `ResearchAgent` (AutonomousAgent) — investigates a single sub-topic and returns a typed `SubTopicFindings{subTopic, summary, sources}`.
- `SynthesisAgent` (AutonomousAgent) — assembles per-sub-topic findings into a coherent report draft; returns a typed `ReportDraft{title, body, sourcesUsed}`.
- `ResearchWorkflow` (Workflow) — a 4-task graph: plan → investigate (fan-out) → synthesise → await review → deliver.
- `ResearchEntity` (EventSourcedEntity) — the report lifecycle and its events.
- `ReportsView` (View) — a read model the UI queries and streams over SSE.
- `ResearchEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/planner-agent.md` — the decomposition instructions and sub-topic count.
- `prompts/research-agent.md` — the investigation depth and source-citing rules.
- `prompts/synthesis-agent.md` — the report structure and citation format.

## What gets validated

- Submitting a query produces a plan and then individual findings appear in `INVESTIGATING`.
- Once all findings arrive, the workflow synthesises and the report appears in `AWAITING_REVIEW` with non-empty body.
- Approving an `AWAITING_REVIEW` report drives it to `DELIVERED` with a non-null delivery timestamp.
- Rejecting a report with feedback drives it back to `NEEDS_REVISION`; the workflow can re-enter the synthesis step.
- The deliver step never runs unless the report is `APPROVED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
