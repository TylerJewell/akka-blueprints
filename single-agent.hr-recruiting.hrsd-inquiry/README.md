# Akka Sample: HRSD Inquiry Agent

A single HR inquiry agent answers employee questions about policies, benefits, leave, and support services — and can submit HR service requests on the employee's behalf. Employee messages are screened for special-category data before they reach the model; policy answers are validated by a guardrail before the agent responds.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a special-category data sanitizer that runs before the agent sees the inquiry, and a `before-agent-response` guardrail that validates every answer for accuracy and policy-reference completeness.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the HR policy catalog lives in-process and the agent's request-submission tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.hr-recruiting.hrsd-inquiry  ~/my-projects/hrsd-inquiry
cd ~/my-projects/hrsd-inquiry
```

(Optional) Edit `SPEC.md` to replace the seeded policy catalog with your organization's actual HR policies, or narrow the agent's scope to a specific domain (e.g., leave-only, benefits-only).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **HrInquiryAgent** — an AutonomousAgent that receives an employee inquiry and returns a typed `InquiryResponse` containing a policy-grounded answer and, optionally, an `HrServiceRequest` ready for submission.
- **InquiryWorkflow** — orchestrates sanitize-wait → answer → optional request-submit per conversation turn.
- **InquiryEntity** — an EventSourcedEntity holding the per-session conversation lifecycle.
- **SpecialCategoryScreener** — a Consumer that subscribes to `InquirySubmitted` events, redacts health conditions, union membership, religious preferences, and other special-category data, and emits `InquiryScreened` back to the entity.
- **InquiryView + InquiryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded policy topics for your organization's actual HR policy identifiers.
- `SPEC.md §5` — extend `InquiryResponse` with organization-specific fields (e.g., `escalationPath`, `policyVersion`, `localeLang`).
- `prompts/hr-inquiry-agent.md` — narrow the agent's role (a healthcare employer would restrict it to FMLA and ADA policy; a retail employer to scheduling and PTO policy).
- `eval-matrix.yaml` — wire a real special-category classifier (e.g., a named-entity recogniser) by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. An employee submits a benefits inquiry → it is screened → answered → the response with a policy citation appears in the UI.
2. The agent returns a response without a policy citation on first try → the `before-agent-response` guardrail rejects it → the agent retries → a properly-cited answer lands.
3. An inquiry containing a health condition is submitted → the screener redacts it → only the redacted form appears in the LLM call log.
4. An employee requests a leave form → the agent produces an `HrServiceRequest` → the workflow submits it → the confirmation number appears on the UI card.

## License

Apache 2.0.
