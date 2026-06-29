# Akka Sample: Post-Incident Review Agent

A single `PIRAgent` walks an incident through three task phases — **GATHER → ASSESS → DRAFT** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a set of phase-specific tools. The operator submits an incident ID and receives a structured `PostIncidentReview` document.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a `before-agent-response` guardrail that screens the final PIR draft before delivery to ensure it meets executive and audit standards, and an `application` human-in-the-loop gate that requires the incident owner to sign off on the review before it is marked complete.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every gather / assess / draft tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.ops-automation.post-incident-reviewer  ~/my-projects/post-incident-reviewer
cd ~/my-projects/post-incident-reviewer
```

(Optional) Edit `SPEC.md` to point at a different incident catalog, a different model provider, or a richer timeline reconstruction tool.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PIRAgent** — one AutonomousAgent declaring three Task constants (`GATHER_EVIDENCE`, `ASSESS_IMPACT`, `DRAFT_REVIEW`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **PIRWorkflow** — runs `gatherStep → assessStep → draftStep → signoffStep`. Each step calls `runSingleTask` and writes the typed result back onto `PIREntity` before the next step starts. `signoffStep` pauses the workflow and waits for an external sign-off command from the incident owner.
- **PIREntity** — an EventSourcedEntity holding the per-review lifecycle (`EvidenceGathered`, `ImpactAssessed`, `ReviewDrafted`, `SignoffRequested`, `SignoffRecorded`, `ReviewComplete`, `ReviewRejected`).
- **GatherTools / AssessTools / DraftTools** — three function-tool classes registered on the agent, one per phase. The `before-agent-response` guardrail fires after the final DRAFT task and before the response is recorded.
- **PIRGuardrail** — the runtime check that backs the response-quality contract. A draft that is missing an executive summary, has no action items, or cites a timeline entry absent from the gathered evidence log is rejected before being written to the entity.
- **PIRSignoffNotifier** — calls the application-layer sign-off integration (webhook or in-process stub) to notify the incident owner when a review is ready for sign-off.
- **PIRView + PIREndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded incident set under `src/main/resources/sample-events/incidents.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/pir-agent.md` — narrow the agent's role (e.g., constrain it to security incidents, to cloud-provider outages, to database-tier failures) by tightening the system prompt and renaming the typed records.
- `SPEC.md §5` — extend the typed outputs (`EvidenceLog`, `ImpactAssessment`, `PostIncidentReview`) with org-specific fields.
- `eval-matrix.yaml` — wire a real completeness check (replace the structural-check stub with a policy rule that validates action-item owners are present) by editing the `G1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. An operator submits an incident ID → `GATHER` runs → `ASSESS` runs → `DRAFT` runs → a typed `PostIncidentReview` is ready for sign-off within ~60 s. Every transition is visible in real time.
2. The PIRGuardrail rejects a draft that is missing a required section (forced via the mock LLM) → the workflow records the rejection → the agent retries within its budget → the pipeline eventually produces a valid draft.
3. The incident owner POSTs a sign-off to `/api/pir/{id}/signoff` → the workflow unpauses → the review transitions to `COMPLETE`.
4. The incident owner rejects a draft (POST to `/api/pir/{id}/signoff` with `approved: false`) → the entity transitions to `REJECTED` → the UI flags the card.

## License

Apache 2.0.
