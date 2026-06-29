# Akka Sample: Medical Pre-Authorization

A single `PreAuthAgent` walks a clinical request through three task phases — **VALIDATE → REVIEW → DETERMINE** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. Protected health information is sanitized at the service boundary before any LLM call, every denial determination is held for a human reviewer before it is finalized, and an on-decision evaluator scores every determination for clinical completeness.

Demonstrates the **sequential-pipeline** coordination pattern wired with three governance mechanisms: a `before-tool-call` guardrail backed by a PHI sanitizer that blocks raw patient identifiers from reaching the LLM, a human-in-the-loop hold on denial determinations, and an `on-decision-eval` evaluator that scores every determination for policy coverage and clinical evidence adequacy.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every validate / review / determine tool is implemented in-process inside the same Akka service using bundled sample clinical guidelines and payer policy files.

## Generate the system

```sh
cp -r ./sequential-pipeline.healthcare.medical-preauth  ~/my-projects/medical-preauth
cd ~/my-projects/medical-preauth
```

(Optional) Edit `SPEC.md` to point at your payer's clinical policy files, adjust the PHI field list, or change the denial-hold timeout.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PreAuthAgent** — one AutonomousAgent declaring three Task constants (`VALIDATE_REQUEST`, `REVIEW_POLICY`, `DETERMINE_OUTCOME`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **PreAuthWorkflow** — runs `validateStep → reviewStep → determineStep → evalStep → (optional) humanHoldStep`. Each step calls `runSingleTask` and writes the typed result back onto `PreAuthEntity` before the next step starts.
- **PreAuthEntity** — an EventSourcedEntity holding the per-request lifecycle (`RequestValidated`, `PolicyReviewed`, `DeterminationWritten`, `EvaluationScored`, `HumanApprovalRequested`, `HumanApprovalReceived`).
- **PhiSanitizer** — the `before-tool-call` guardrail. Strips or hashes PHI fields (MRN, DOB, full name, SSN) from every outbound tool call payload before the LLM sees them. Emits a `SanitizationApplied` audit event for every field removed.
- **ValidateTools / ReviewTools / DetermineTools** — three function-tool classes registered on the agent, one per phase. The phase-gate guardrail enforces that each tool is only callable in its own phase.
- **PhaseGuardrail** — the runtime check that backs the dependency contract. A tool call referencing a phase whose precondition has not been recorded on the entity is rejected before the tool runs.
- **ClinicalCompletenessScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `DeterminationWritten` and emits a 1–5 score.
- **DenialHoldStep** — a human-in-the-loop pause on any determination whose outcome is `DENIED`. The workflow waits up to 72 hours for a reviewer acknowledgement; if none arrives, the determination is escalated.
- **PreAuthView + PreAuthEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded request set under `src/main/resources/sample-events/requests.jsonl` to match your payer's procedure-code coverage.
- `SPEC.md §4` and `prompts/pre-auth-agent.md` — tighten the agent's role (e.g., restrict it to oncology, musculoskeletal, or behavioral health procedure codes) by editing the system prompt and the `ClinicalPolicy` and `PolicyEvidence` records.
- `SPEC.md §5` — extend the typed outputs (`ValidationResult`, `PolicyReview`, `Determination`) with payer-specific fields. The phase-gating guardrail does not need editing — it checks recorded-phase preconditions, not field shapes.
- `eval-matrix.yaml` — swap the deterministic `ClinicalCompletenessScorer` for a real clinical-guidelines vector-similarity check by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A clinician submits a pre-auth request for a covered procedure → `VALIDATE` runs → `REVIEW` runs → `DETERMINE` runs → an `APPROVED` determination with a completeness score of 5 lands in the UI within ~90 s.
2. A pre-auth request whose submitted MRN appears in any outbound tool call — before PHI sanitization is applied — is flagged. After `PhiSanitizer` runs, the MRN is replaced with a hashed token; the `SanitizationApplied` event records the field and the hash. The LLM never sees raw patient identifiers.
3. A `DENIED` determination triggers `DenialHoldStep`. The UI shows the request stuck in `PENDING_HUMAN_REVIEW`. A reviewer clicks **Acknowledge** → the workflow resumes; the determination finalizes as `EVALUATED`.
4. An LLM-generated determination that cites a procedure code absent from the validated request is scored 1 by `ClinicalCompletenessScorer` and flagged for immediate human review, regardless of the approval/denial outcome.

## License

Apache 2.0.
