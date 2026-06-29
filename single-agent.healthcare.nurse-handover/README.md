# Akka Sample: Nurse Handover

A single clinical-handover agent reads an incoming shift report, maps every active patient against a configurable check list, and returns a structured `HandoverSummary` — a per-patient status with outstanding tasks, risk flags, and a sign-off prompt. PHI is stripped before the agent ever sees the report. A clinician must explicitly sign off each summary before it is sealed.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a PHI sanitizer that runs before the agent reads any patient data, and a human-in-the-loop sign-off gate that holds the handover open until a named clinician confirms it.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — patient data is seeded in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.healthcare.nurse-handover  ~/my-projects/nurse-handover
cd ~/my-projects/nurse-handover
```

(Optional) Edit `SPEC.md` to point at a different ward checklist (e.g., switch from a general medicine ward template to an ICU template or a paediatric template).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **HandoverAgent** — an AutonomousAgent that accepts a ward checklist plus a sanitized shift report as a task attachment and returns a typed `HandoverSummary`.
- **HandoverWorkflow** — orchestrates sanitize-wait → summarize → await-signoff per submitted handover.
- **HandoverEntity** — an EventSourcedEntity holding the per-handover lifecycle.
- **ReportSanitizer** — a Consumer that subscribes to `ReportSubmitted` events, redacts PHI into a `SanitizedReport`, and emits `ReportSanitized` back to the entity.
- **HandoverView + HandoverEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded ward checklist for your own (the JSONL file under `src/main/resources/sample-events/ward-checklists.jsonl` after generation).
- `SPEC.md §5` — extend `HandoverSummary` with ward-specific fields (e.g., `isolationStatus`, `fallRisk`, `allergiesActive`).
- `prompts/handover-agent.md` — narrow the agent's role (an ICU deployer would constrain it to ventilator and medication weaning checks; a paediatric deployer to weight-adjusted dosing reminders).
- `eval-matrix.yaml` — wire a real PHI redactor (e.g., a presidio-style library) by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A clinician submits a shift report → it is sanitized → summarized → the handover summary appears in the UI awaiting sign-off.
2. A clinician signs off the handover → the entity transitions to `SIGNED_OFF`; a second sign-off attempt is rejected as a no-op.
3. PHI strings submitted in the shift report never appear in the LLM call log; only the redacted form does.
4. A report that omits a patient the checklist requires triggers a `MissingPatientFlag` in the summary.

## License

Apache 2.0.
