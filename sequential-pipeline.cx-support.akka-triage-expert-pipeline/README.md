# Akka Sample: Triage + Expert Multi-Agent Workflow

A workflow invokes two distinct agents as child workflows: a **TriageAgent** gathers customer information and produces a structured summary, then an **ExpertAgent** uses that summary to compose a final recommendation delivered to the customer. The typed handoff between the two agents is the only path information travels between phases.

Demonstrates the **sequential-pipeline** coordination pattern with three governance mechanisms wired around the two-agent handoff: a `before-agent-response` guardrail that reviews the ExpertAgent's output before it reaches the customer, a PII sanitizer that scrubs customer data between the triage and expert phases, and an `on-decision-eval` evaluator that scores every expert recommendation for relevance and grounding.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Both agents and all supporting tools run in-process inside the same Akka service. No external CRM or ticketing system is required.

## Generate the system

```sh
cp -r ./sequential-pipeline.cx-support.akka-triage-expert-pipeline  ~/my-projects/triage-expert
cd ~/my-projects/triage-expert
```

(Optional) Edit `SPEC.md` to change the seeded issue categories in `src/main/resources/sample-events/issues.jsonl`, point at a different model provider, or extend the expert knowledge-base articles.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build`. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TriageAgent** — an AutonomousAgent with two Task constants (`GATHER_CUSTOMER_INFO`, `SUMMARIZE_ISSUE`). The workflow calls them in order, feeding the gathered info forward as the summarize task's context. The agent only has access to customer-facing intake tools during triage.
- **ExpertAgent** — a second AutonomousAgent with one Task constant (`COMPOSE_RECOMMENDATION`). It receives the sanitized triage summary as its entire instruction context and returns a typed `Recommendation`. It has access to knowledge-base lookup tools.
- **SupportCaseWorkflow** — one workflow per support case. Steps: `gatherStep → summarizeStep → sanitizeStep → recommendStep → evalStep`. The `sanitizeStep` runs the `PiiSanitizer` before handing off to the ExpertAgent. The `recommendStep` is wrapped by the `RecommendationGuardrail` before the result is written onto the entity.
- **SupportCaseEntity** — an EventSourcedEntity holding the per-case lifecycle (`CustomerInfoGathered`, `IssueSummarized`, `SummaryPii­Sanitized`, `RecommendationComposed`, `EvaluationScored`, `GuardrailBlocked`).
- **IntakeTools / KnowledgeBaseTools** — two function-tool classes, one per agent. `IntakeTools` queries the in-process customer-intake data; `KnowledgeBaseTools` looks up knowledge articles.
- **PiiSanitizer** — a deterministic sanitizer that scrubs names, emails, account IDs, and phone numbers from the triage summary before it is handed to the ExpertAgent.
- **RecommendationGuardrail** — a `before-agent-response` guardrail registered on `ExpertAgent`. It checks the composed recommendation against a disallowed-content policy before the result is recorded on the entity.
- **RecommendationScorer** — a deterministic, rule-based on-decision evaluator that runs after `RecommendationComposed` and emits a 1–5 grounding score.
- **SupportCaseView + SupportCaseEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded issue categories under `src/main/resources/sample-events/issues.jsonl` to match your demo's product surface.
- `SPEC.md §4` and `prompts/triage-agent.md` — narrow the TriageAgent's intake focus (e.g., billing-only, account access, technical troubleshooting) by editing the system prompt and renaming the typed records (`CustomerInfo`, `IssueSummary`).
- `SPEC.md §4` and `prompts/expert-agent.md` — extend the ExpertAgent with richer knowledge-base articles by editing the tool's backing JSON files in `src/main/resources/knowledge-base/`.
- `eval-matrix.yaml` — replace the deterministic `RecommendationScorer` stub with a semantic-similarity check against a reference answer corpus by updating the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer submits an issue → triage completes → PII is scrubbed → the ExpertAgent composes a recommendation → an eval score appears on the card within ~90 s. Every phase transition is visible in real time.
2. The ExpertAgent composes a recommendation that contains a disallowed phrase (forced via the mock LLM) → `RecommendationGuardrail` blocks it → a `GuardrailBlocked` event lands on the entity → the agent retries within its iteration budget → the case completes with a compliant recommendation.
3. The triage summary passed to the ExpertAgent contains no PII — verified by asserting that names and account IDs from the customer intake do not appear in the sanitized summary text.
4. Every recommendation emitted has a grounding score visible on the same UI card; recommendations that cite no knowledge-base article score ≤ 2 and are flagged for review.

## License

Apache 2.0.
