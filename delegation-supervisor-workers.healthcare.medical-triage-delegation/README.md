# Akka Sample: Medical Agent Delegation

A triage agent receives an incoming medical question, delegates it to specialist sub-agents (symptoms, medications, care-pathway), and synthesises a single patient-safe response. Demonstrates the **delegation-supervisor-workers** coordination pattern with healthcare-grade governance controls.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound question stream and the specialist knowledge tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.healthcare.medical-triage-delegation  ~/my-projects/medical-triage-delegation
cd ~/my-projects/medical-triage-delegation
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or specialist roster.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TriageCoordinator** — AutonomousAgent that classifies the question, decomposes it into specialist queries, and synthesises a final patient-safe response.
- **SymptomsSpecialist** — AutonomousAgent that interprets symptom descriptions and returns a differential summary.
- **MedicationsSpecialist** — AutonomousAgent that checks drug interactions, contraindications, and dosage guidance.
- **CarePathwaySpecialist** — AutonomousAgent that recommends the appropriate care pathway (self-care, GP, urgent care, emergency).
- **TriageWorkflow** — Workflow that fans work out to all three specialists in parallel, then calls TriageCoordinator for synthesis.
- **CaseEntity** — EventSourcedEntity holding the case lifecycle.
- **CaseView** — projection the UI streams via SSE.
- **TriageEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the question topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `MedicalCase` record fields (e.g., add `urgencyScore`).
- `prompts/triage-coordinator.md` — tighten the safety guardrail language.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a live drug-interaction API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a medical question → case enters `TRIAGING`, then `IN_REVIEW`, then `RESPONDED`.
2. Specialist failure → if a specialist times out, the case enters `DEGRADED` with whichever partial outputs exist; the response notes missing coverage.
3. Safety guardrail blocks a response containing diagnosis claims; the case enters `BLOCKED`.
4. A human compliance reviewer is notified (hotl event); the UI shows the pending review flag.

## License

Apache 2.0.
