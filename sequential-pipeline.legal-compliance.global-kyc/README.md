# Akka Sample: Global KYC Agent

A single `KycAgent` walks a customer identity submission through three task phases — **COLLECT → VERIFY → DECIDE** — wired by explicit task dependencies. Each phase has its own typed input, typed output, and a focused tool set. The user submits an applicant profile and receives a structured `KycDecision`.

Demonstrates the **sequential-pipeline** coordination pattern wired with three governance mechanisms: a PII sanitizer that scrubs identity data before it leaves the system boundary, a human-in-the-loop review gate that holds adverse decisions for compliance officer sign-off, and an `on-decision-eval` evaluator that scores every emitted decision for documentation completeness and regulatory rule coverage.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Every collection, verification, and decision tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.legal-compliance.global-kyc  ~/my-projects/global-kyc
cd ~/my-projects/global-kyc
```

(Optional) Edit `SPEC.md` to point at a different jurisdiction rule set, a different model provider, or a richer set of document verification tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **KycAgent** — one AutonomousAgent declaring three Task constants (`COLLECT_DOCUMENTS`, `VERIFY_IDENTITY`, `RENDER_DECISION`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **KycPipelineWorkflow** — runs `collectStep → verifyStep → decideStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `KycCaseEntity` before the next step starts.
- **KycCaseEntity** — an EventSourcedEntity holding the per-case lifecycle (`DocumentsCollected`, `IdentityVerified`, `DecisionRendered`, `DecisionEvaluated`).
- **CollectTools / VerifyTools / DecideTools** — three function-tool classes registered on the agent, one per phase. The `before-sanitizer` hook in `PiiSanitizer` scrubs raw identity fields before they propagate beyond the collect phase.
- **PiiSanitizer** — the runtime sanitizer that redacts or tokenises PII fields (full name, date of birth, document number) in the outbound record before writing to the view. The raw values stay in the entity event log under access control; the view exposes only tokenised references.
- **HitlReviewGate** — the human-in-the-loop component invoked by `decideStep` when `KycDecision.outcome == DECLINE`. Pauses the workflow and emits a `ReviewRequested` event; a compliance officer posts to `/api/cases/{id}/review` to resume with an `APPROVED` or `OVERRIDDEN_APPROVE` outcome.
- **DecisionScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `DecisionRendered` and emits a 1–5 score measuring documentation completeness and rule coverage.
- **KycView + KycEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded applicant profiles under `src/main/resources/sample-events/applicants.jsonl` to reflect your demo geography and document types.
- `SPEC.md §4` and `prompts/kyc-agent.md` — narrow the agent's role (e.g., restrict to a single jurisdiction's rules) by tightening the system prompt and renaming rule sets.
- `SPEC.md §5` — extend the typed outputs (`DocumentSet`, `VerificationResult`, `KycDecision`) with jurisdiction-specific fields. The phase-gating sanitizer does not need editing — it checks recorded-phase preconditions and field names, not payload shapes.
- `eval-matrix.yaml` — wire a real document-authenticity evaluator (replace the deterministic stub with an OCR-confidence check) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A compliance officer submits an applicant profile → `COLLECT` runs → `VERIFY` runs → `DECIDE` runs → a typed `KycDecision` with outcome `PASS` lands in the UI within ~60 s. Every transition is visible in real time.
2. A DECLINE decision triggers the HITL gate; the case pauses at `PENDING_REVIEW`; after a compliance officer posts an approval, the case resumes and reaches `DECIDED`.
3. Every `KycDecision` emitted has an on-decision eval score visible on the same UI card; decisions that cite fewer than two verification rules receive a score ≤ 2 and are flagged.
4. PII fields in the UI card are tokenised (e.g., `DOC-****-7843`) — the sanitizer ran; the raw values are absent from every view row and SSE event.

## License

Apache 2.0.
