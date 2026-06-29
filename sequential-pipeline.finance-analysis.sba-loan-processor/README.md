# Akka Sample: Small Business Loan Agent

A single `LoanUnderwritingAgent` walks a loan application through four task phases — **INTAKE → UNDERWRITE → DECISION → REPORT** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a set of phase-specific tools. The applicant's data is sanitized of PII and protected attributes before the agent sees it; a human loan officer reviews every approval or denial before it takes effect; and an on-decision evaluator scores every decision for fair-lending compliance.

Demonstrates the **sequential-pipeline** coordination pattern wired with four governance mechanisms: a `before-call` PII sanitizer that strips applicant identifiers from agent context, a `before-call` special-category sanitizer that masks protected attributes (race, ethnicity, sex, age, marital status), a human-in-the-loop approval gate that holds every decision pending a loan-officer sign-off, and an `on-decision-eval` evaluator that scores every underwriting decision for fair-lending signal.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Every underwriting tool — credit-score lookup, cash-flow analysis, collateral valuation — is implemented in-process using seeded sample data.

## Generate the system

```sh
cp -r ./sequential-pipeline.finance-analysis.sba-loan-processor  ~/my-projects/sba-loan-processor
cd ~/my-projects/sba-loan-processor
```

(Optional) Edit `SPEC.md` to change the seeded applicant set, swap in a different model provider, or adjust the fair-lending scoring weights.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **LoanUnderwritingAgent** — one AutonomousAgent declaring four Task constants (`INTAKE_APPLICATION`, `UNDERWRITE_APPLICATION`, `MAKE_DECISION`, `GENERATE_REPORT`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **LoanPipelineWorkflow** — runs `intakeStep → underwriteStep → decisionStep → hitlStep → evalStep → reportStep`. Each agent step calls `runSingleTask` and writes the typed result back onto `LoanApplicationEntity` before the next step starts. `hitlStep` pauses the workflow until a loan officer submits a review.
- **LoanApplicationEntity** — an EventSourcedEntity holding the per-application lifecycle from `SUBMITTED` through `OFFICER_APPROVED` or `OFFICER_DENIED`.
- **IntakeTools / UnderwriteTools / DecisionTools / ReportTools** — four function-tool classes registered on the agent, one per phase. A `before-tool-call` guardrail enforces that each tool is only callable in its own phase.
- **PiiSanitizer** — a `before-call` sanitizer that strips direct applicant identifiers (SSN, full name, date of birth, address) before the agent's context is assembled.
- **ProtectedAttributeSanitizer** — a `before-call` sanitizer that masks special-category fields (race, ethnicity, sex, age, marital status, national origin) so the agent's underwriting output cannot be influenced by them.
- **PhaseGuardrail** — enforces that each tool class is only callable when its phase's preconditions hold on the entity.
- **FairLendingScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `DecisionMade` and emits a 1–5 fairness signal score.
- **LoanApplicationView + LoanEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded applicant profiles under `src/main/resources/sample-data/applications/*.json` to fit your demo audience.
- `SPEC.md §4` and `prompts/loan-underwriting-agent.md` — narrow the agent's role to a specific loan product (e.g. SBA 7(a), SBA 504, microloans) by tightening the system prompt and adjusting the decision criteria.
- `SPEC.md §5` — extend the typed outputs (`CreditProfile`, `UnderwritingAnalysis`, `LoanDecision`) with lender-specific fields such as collateral type or guarantor details.
- `eval-matrix.yaml` — wire a real fair-lending model (replace the deterministic stub with an adverse-impact ratio check against a historical approval corpus) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A loan officer submits a seeded application → `INTAKE` runs → `UNDERWRITE` runs → `DECISION` runs → an underwriting recommendation lands in the UI and a human-review request appears in the officer queue within ~90 s.
2. An attempt to call a `DecisionTools` method during the `INTAKE` phase (forced via mock LLM) is blocked by the `PhaseGuardrail`; the rejection is logged on the entity and the pipeline self-corrects inside its iteration budget.
3. An application with a protected attribute present in the raw input arrives at the agent with those fields masked; the agent's underwriting rationale contains no reference to the masked attributes.
4. Every `LoanDecision` emitted has an on-decision fair-lending eval score visible on the same UI card; decisions where the denial rate pattern deviates from baseline receive a score ≤ 2 and are flagged for supervisory review.
5. A human loan officer approves or denies the recommendation in the officer queue; the entity transitions to `OFFICER_APPROVED` or `OFFICER_DENIED` and the SSE stream updates all listening clients.

## License

Apache 2.0.
