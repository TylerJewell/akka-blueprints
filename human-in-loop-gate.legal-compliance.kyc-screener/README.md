# Akka Sample: KYC Screener

An entity file is assembled by a screener agent, source documents are reviewed for adverse findings, and a compliance officer approves or rejects the verdict through an application-level human gate before any case is closed.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.legal-compliance.kyc-screener  ~/my-projects/kyc-screener
cd ~/my-projects/kyc-screener
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `ScreenerAgent` (AutonomousAgent) — assembles an entity file from supplied source documents and produces a typed `ScreeningResult{entityId, findings, recommendation}`.
- `VerdictAgent` (AutonomousAgent) — packages the approved screening result into a typed `ClosedCase{caseId, disposition, closedAt}`.
- `ScreeningWorkflow` (Workflow) — a 3-task graph: screen → await compliance-officer approval → close.
- `CaseEntity` (EventSourcedEntity) — the case lifecycle and its events.
- `CasesView` (View) — a read model the UI queries and streams over SSE.
- `ScreeningEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/screener-agent.md` — the screening instructions and source-citation rules.

## What gets validated

- Submitting an entity id with documents triggers a screening result in `SCREENED` with non-empty findings.
- Approving a `SCREENED` case drives it to `CLOSED` with disposition `APPROVED`.
- Rejecting a `SCREENED` case drives it to terminal `REJECTED` with the reason shown.
- The close step never runs unless the case is `OFFICER_APPROVED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
