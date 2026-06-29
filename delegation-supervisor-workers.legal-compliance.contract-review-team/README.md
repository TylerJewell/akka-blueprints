# Akka Sample: Contract Assistant (Multi-Agent)

A contract-review supervisor delegates clause analysis to a `ClauseAnalyst`, risk scoring to a `RiskScorer`, and redline drafting to a `Redliner` — all running in parallel — then merges their outputs into one reviewed contract package. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded legal-sector governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound contract queue and legal-domain tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.legal-compliance.contract-review-team  ~/my-projects/contract-assistant
cd ~/my-projects/contract-assistant
```

(Optional) Edit `SPEC.md` to change the contract queue topics the simulator drips, or adjust which clause types the workers handle.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReviewSupervisor** — AutonomousAgent that decomposes a contract into a review plan and merges all worker outputs into a final reviewed package.
- **ClauseAnalyst** — AutonomousAgent that identifies and categorises key clauses.
- **RiskScorer** — AutonomousAgent that assigns risk levels and flags problematic language.
- **Redliner** — AutonomousAgent that drafts redline suggestions for high-risk clauses.
- **ReviewWorkflow** — Workflow that fans work to ClauseAnalyst, RiskScorer, and Redliner in parallel, then calls ReviewSupervisor for consolidation.
- **ContractReviewEntity** — EventSourcedEntity holding the contract review lifecycle.
- **ContractSanitizer** — legal-sector content filter applied before any worker receives contract text.
- **ReviewView** — projection the UI streams via SSE.
- **ReviewEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the contract samples the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `ContractReview` record fields (e.g., add `jurisdictionCode`).
- `prompts/clause-analyst.md` — narrow the analyst to specific contract types (NDA, MSA, SoW).
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real document store.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a contract → review enters `QUEUED`, then `IN_REVIEW`, then `AWAITING_APPROVAL`.
2. Workers fail-fast → if any worker times out, the review enters `DEGRADED` with whichever partial outputs exist.
3. Legal-sector sanitizer rejects a contract with prohibited content → review enters `REJECTED` before any agent sees the text.
4. A lawyer approves the redlines → review transitions to `APPROVED` and the package is locked.

## License

Apache 2.0.
