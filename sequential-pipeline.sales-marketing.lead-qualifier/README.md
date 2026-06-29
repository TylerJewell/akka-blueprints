# Akka Sample: AI-Driven Lead Qualifier

An `InquiryAgent` walks a raw sales inquiry through three task phases — **CAPTURE → QUALIFY → ENRICH** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a phase-specific set of tools. The user submits an inquiry and receives a qualified `LeadRecord` ready for CRM entry.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a `before-tool-call` guardrail that validates every CRM write for schema conformance and scope before execution, and a `pii` sanitizer that strips or masks personal contact fields before they travel outside the service boundary.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs entirely in-process — every capture, qualify, and enrich tool is implemented inside the same Akka service without an external CRM or ERPNext instance.

## Generate the system

```sh
cp -r ./sequential-pipeline.sales-marketing.lead-qualifier  ~/my-projects/lead-qualifier
cd ~/my-projects/lead-qualifier
```

(Optional) Edit `SPEC.md` to point at a different set of seeded inquiry forms, a different model provider, or a stricter qualification scoring threshold.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **InquiryAgent** — one AutonomousAgent declaring three Task constants (`CAPTURE_INQUIRY`, `QUALIFY_LEAD`, `ENRICH_CRM`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **LeadPipelineWorkflow** — runs `captureStep → qualifyStep → enrichStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `LeadEntity` before the next step starts.
- **LeadEntity** — an EventSourcedEntity holding the per-lead lifecycle (`InquiryCaptured`, `QualificationScored`, `CrmEntryWritten`, `EvaluationScored`).
- **CaptureTools / QualifyTools / EnrichTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that CRM-write tools are only callable after qualification has been recorded.
- **CrmWriteGuardrail** — the runtime check that backs the CRM dependency contract. A tool call that would write to CRM before the lead has been qualified is rejected before the tool runs.
- **PiiSanitizer** — intercepts outbound payloads before any tool call that crosses the service boundary (currently the CRM enrich tools) and masks or removes PII fields per the configured policy.
- **QualityScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `CrmEntryWritten` and emits a 1–5 data-quality score.
- **LeadView + LeadEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded inquiry set under `src/main/resources/sample-events/inquiries.jsonl` to fit your demo product or vertical.
- `SPEC.md §4` and `prompts/inquiry-agent.md` — narrow the agent's role (e.g., restrict it to B2B SaaS leads, to inbound event-registration inquiries, to channel-partner referrals) by tightening the system prompt and renaming the typed records (`InquiryForm`, `LeadScore`, `CrmEntry`).
- `SPEC.md §5` — extend the typed outputs (`InquiryForm`, `LeadScore`, `CrmEntry`) with vertical-specific fields. The CRM-write guardrail does not need editing — it checks recorded-qualification preconditions, not field shapes.
- `eval-matrix.yaml` — wire a real CRM schema validator (replace the deterministic stub with a live ERPNext field-validation call) by editing the `G1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits an inquiry → `CAPTURE` runs → `QUALIFY` runs → `ENRICH` runs → a typed `CrmEntry` lands in the UI within ~60 s. Every transition is visible in real time.
2. An enrich tool is called before qualification is recorded (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries in-phase → the pipeline completes correctly.
3. Every `CrmEntry` emitted has an on-decision data-quality score visible on the same UI card; entries with missing required CRM fields receive a score ≤ 2 and are flagged.
4. Personal contact fields in the inquiry (email, phone, full name) are masked in any payload that leaves the capture phase; the qualify and enrich phases see only anonymised identifiers.

## License

Apache 2.0.
