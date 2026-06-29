# Akka Sample: Inbound Lead Qualification & Research Agent

An inbound lead form submission triggers a three-phase pipeline — **ENRICH → QUALIFY → NOTIFY** — driven by one `LeadAgent`. Each phase has its own typed input, typed output, and a scoped set of function tools. The form submitter's data is PII-sanitized before any enrichment tool runs; a Slack notification carrying the qualification score is gated behind a pre-tool guardrail; and every qualification decision is sampled by an on-decision evaluator so the scoring model's accuracy can be monitored over time.

Demonstrates the **sequential-pipeline** coordination pattern wired with three governance mechanisms: a `before-tool-call` guardrail that blocks the Slack write until the qualified lead record is complete, a PII sanitizer that strips raw form fields before any enrichment call, and an `on-decision-eval` evaluator that samples qualification scores for downstream accuracy review.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A valid Slack credential (bot token with `chat:write` scope). Supply it using the same key-sourcing options listed above — env var, env file, secrets URI, or type-once in session. Akka never writes the credential value to disk. The integration label for this blueprint is **Slack Adapter**.

## Generate the system

```sh
cp -r ./sequential-pipeline.sales-marketing.lead-qualification-pipeline  ~/my-projects/lead-qualification
cd ~/my-projects/lead-qualification
```

(Optional) Edit `SPEC.md` to point at a different qualification scoring rubric, a different Slack channel, or a richer firmographic enrichment tool set.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **LeadAgent** — one AutonomousAgent declaring three Task constants (`ENRICH_LEAD`, `QUALIFY_LEAD`, `NOTIFY_SALES`); the workflow runs them in order, feeding each typed output forward as the next task's instruction context.
- **LeadPipelineWorkflow** — runs `enrichStep → qualifyStep → notifyStep → evalStep`. Each step calls `runSingleTask`, writes the typed result back onto `LeadEntity`, then advances.
- **LeadEntity** — an EventSourcedEntity holding the per-lead lifecycle (`LeadCreated`, `EnrichmentCompleted`, `QualificationScored`, `NotificationSent`, `EvaluationRecorded`).
- **EnrichTools / QualifyTools / NotifyTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that Slack write tools are only callable once the qualified lead record is present.
- **PiiSanitizer** — a system-level sanitizer that strips raw PII fields (name, email, company) from the enrichment task's instruction context before the enrichment tools can log or forward them.
- **SlackGuardrail** — the runtime `before-tool-call` check that prevents `postLeadToSlack` from firing until `QualificationScored` has been recorded on the entity.
- **QualificationEvaluator** — deterministic, rule-based on-decision evaluator that runs immediately after `QualificationScored` and records whether the score is consistent with the enrichment evidence.
- **LeadView + LeadEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded lead set under `src/main/resources/sample-events/leads.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/lead-agent.md` — narrow the agent's scoring rubric (e.g., constrain it to SaaS buyers, to enterprise accounts above $10 M ARR, or to a specific geography) by tightening the system prompt and adjusting the `QualificationScore` record's `tier` enum values.
- `SPEC.md §5` — extend the typed records (`FirmographicProfile`, `QualificationScore`) with industry-specific fields. The guardrail does not need editing — it checks recorded-phase preconditions, not field shapes.
- `eval-matrix.yaml` — wire a real sampling evaluator (replace the deterministic rule stub with a statistical sampling check) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A lead form is submitted → `ENRICH` runs → `QUALIFY` runs → `NOTIFY` runs → a scored `QualificationScore` and a Slack notification land within ~60 s. Every transition is visible in real time.
2. The `postLeadToSlack` tool is attempted before `QualificationScored` is recorded (forced via the mock LLM) → `SlackGuardrail` rejects the call → the workflow records the rejection event → the agent retries in-phase → the pipeline completes correctly.
3. Every `QualificationScore` emitted has an on-decision eval record visible on the same UI card; scores whose enrichment evidence does not support the stated tier receive a low confidence flag.
4. Each task receives only its own typed inputs; the ENRICH task does not see the Slack message template, and the NOTIFY task does not see raw firmographic API responses — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
