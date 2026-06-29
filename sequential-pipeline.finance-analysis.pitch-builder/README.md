# Akka Sample: Pitch Builder

A single `PitchAgent` walks a deal target through three task phases — **RESEARCH → COMPARABLES → DRAFT** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The user submits a target company name and receives a structured `Pitchbook`.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a PII sanitizer that redacts counterparty identifiers from raw research inputs before the agent sees them, and a `before-agent-response` guardrail that validates citations and financial figures before the draft pitchbook is returned to the caller.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every research, comparables, and draft tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.finance-analysis.pitch-builder  ~/my-projects/pitch-builder
cd ~/my-projects/pitch-builder
```

(Optional) Edit `SPEC.md` to point at a different target-company seed list, a different model provider, or a richer set of comparables tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PitchAgent** — one AutonomousAgent declaring three Task constants (`RESEARCH_TARGET`, `RUN_COMPARABLES`, `DRAFT_PITCHBOOK`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **PitchbookPipelineWorkflow** — runs `researchStep → comparablesStep → draftStep → validationStep`. Each step calls `runSingleTask` and writes the typed result back onto `PitchbookEntity` before the next step starts.
- **PitchbookEntity** — an EventSourcedEntity holding the per-pitchbook lifecycle (`TargetResearched`, `ComparablesBuilt`, `DraftWritten`, `ValidationScored`).
- **ResearchTools / ComparablesTools / DraftTools** — three function-tool classes registered on the agent, one per phase. The PII sanitizer strips counterparty identifiers from research data before any tool output reaches the agent's context.
- **PiiSanitizer** — the pre-processing hook that backs the data-cleanliness contract. Research outputs are scanned for known PII patterns (counterparty contact names, personal email addresses, direct phone numbers) and redacted before the agent task context is assembled.
- **CitationValidator** — deterministic, rule-based before-agent-response validator that runs after `DraftWritten` and checks every financial figure and citation in the pitchbook for provenance against the collected research and comparables.
- **PitchbookView + PitchbookEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded target list under `src/main/resources/sample-events/targets.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/pitch-agent.md` — narrow the agent's role (e.g., constrain it to technology sector targets, to mid-market buyout mandates, to public-equity coverage initiations) by tightening the system prompt and renaming the typed records (`ResearchPack`, `CompsTable`, `Pitchbook`).
- `SPEC.md §5` — extend the typed outputs (`ResearchPack`, `CompsTable`, `Pitchbook`) with sector-specific fields. The PII sanitizer does not need editing — it checks field values against a configurable pattern registry.
- `eval-matrix.yaml` — wire a real citation validator (replace the deterministic stub with a vector-similarity check against the collected research corpus) by editing the `H1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a target → `RESEARCH` runs → `COMPARABLES` runs → `DRAFT` runs → a typed `Pitchbook` lands in the UI within ~60 s. Every transition is visible in real time.
2. A research output containing a real counterparty contact name is sanitized before reaching the agent; the contact name does not appear anywhere in the final pitchbook or in the logged task context.
3. Every `Pitchbook` returned by the agent has a citation-validation score visible on the same UI card; drafts that cite figures absent from the recorded `CompsTable` receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the RESEARCH task does not see the draft instructions, and the DRAFT task does not see raw research scrapes — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
