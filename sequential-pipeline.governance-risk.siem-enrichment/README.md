# Akka Sample: SIEM Alert Enrichment with MITRE ATT\&CK and Zendesk

A single `AlertEnrichmentAgent` walks each incoming SIEM alert through three task phases — **FETCH → ENRICH → TRIAGE** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The operator submits an alert and receives a structured `TriageTicket` opened in Zendesk.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a `before-tool-call` guardrail that blocks the Zendesk ticket-creation tool until enrichment has been recorded, and an `on-incident-reporter` evaluator that scores every emitted triage ticket for ATT\&CK coverage and confidence completeness.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every FETCH / ENRICH / TRIAGE tool is implemented in-process, with Zendesk calls stubbed by a deterministic in-process adapter.

## Generate the system

```sh
cp -r ./sequential-pipeline.governance-risk.siem-enrichment  ~/my-projects/siem-enrichment
cd ~/my-projects/siem-enrichment
```

(Optional) Edit `SPEC.md` to point at a different alert corpus, a different model provider, or a richer MITRE ATT\&CK technique registry.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AlertEnrichmentAgent** — one AutonomousAgent declaring three Task constants (`FETCH_ALERT`, `ENRICH_ALERT`, `CREATE_TICKET`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **AlertEnrichmentWorkflow** — runs `fetchStep → enrichStep → ticketStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `AlertEntity` before the next step starts.
- **AlertEntity** — an EventSourcedEntity holding the per-alert lifecycle (`AlertReceived`, `FetchCompleted`, `EnrichmentProduced`, `TicketCreated`, `EvaluationScored`).
- **FetchTools / EnrichTools / TicketTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that ticket-creation tools are only callable once enrichment has been recorded.
- **TicketScopeGuardrail** — the runtime check that backs the dependency contract. A tool call attempting to open a Zendesk ticket when the enrichment phase has not yet recorded its output is rejected before the tool runs.
- **TriageQualityScorer** — deterministic, rule-based on-incident evaluator that runs immediately after `TicketCreated` and emits a 1–5 score for ATT\&CK coverage and confidence completeness.
- **AlertView + AlertEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded alert set under `src/main/resources/sample-events/alerts.jsonl` to fit your demo environment or threat model.
- `SPEC.md §4` and `prompts/alert-enrichment-agent.md` — narrow the agent's role (e.g., constrain it to cloud workload threats, to identity-based attacks, to ransomware campaigns) by tightening the system prompt and renaming the typed records (`EnrichedAlert`, `AttackPattern`, `TriageTicket`).
- `SPEC.md §5` — extend the typed outputs (`AlertDetail`, `EnrichedAlert`, `TriageTicket`) with environment-specific fields. The ticket-scope guardrail does not need editing — it checks recorded-enrichment preconditions, not field shapes.
- `eval-matrix.yaml` — wire a real ATT\&CK confidence evaluator (replace the deterministic stub with a lookup against the MITRE ATT\&CK STIX bundle) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. An operator submits a SIEM alert → `FETCH` runs → `ENRICH` runs → `TRIAGE` runs → a typed `TriageTicket` lands in the UI within ~60 s. Every transition is visible in real time.
2. A ticket-creation tool is called before `EnrichmentProduced` has been recorded (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries in-phase → the pipeline completes correctly.
3. Every `TriageTicket` emitted has an on-incident eval score visible on the same UI card; tickets whose attack-pattern list references no confirmed MITRE technique receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the FETCH task does not see the ticket instructions, and the TRIAGE task does not see raw alert fields — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
