# Akka Sample: Passport Renewal Coordinator

A `RenewalCoordinatorAgent` plans the end-to-end passport renewal flow per applicant, routes each step to a `DocumentReviewAgent`, tokenizes PII fields before any LLM exposure, halts when required documents are missing, and gates the final submission to the issuing authority on a caseworker approval. Demonstrates the **planner-executor** coordination pattern with embedded PII sanitization and human-in-the-loop governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Agency endpoints are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.public-sector.passport-renewal-coordinator  ~/my-projects/passport-renewal-coordinator
cd ~/my-projects/passport-renewal-coordinator
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RenewalCoordinatorAgent** — AutonomousAgent that plans the renewal steps on a renewal ledger and decides the next action each workflow iteration.
- **DocumentReviewAgent** — AutonomousAgent that examines the applicant's document set for completeness and validity using seeded fixtures.
- **RenewalWorkflow** — Workflow with a plan → check-halt → propose → pii-tokenize → document-review → caseworker-wait → agency-submit loop, with missing-docs and reject branches.
- **ApplicationEntity** — EventSourcedEntity holding the application lifecycle, renewal ledger, document status, caseworker decision, and the final outcome.
- **CaseworkerControlEntity** — EventSourcedEntity holding the per-application caseworker decision (approve / reject with a reason).
- **ApplicationQueue** — EventSourcedEntity that is the audit log of submitted applications.
- **ApplicationView** — projection used by the UI.
- **ApplicationConsumer** — Consumer that starts a workflow per submission.
- **RequestSimulator** — TimedAction that drips a sample application every 90 s so the App UI is not empty when first loaded.
- **StaleApplicationMonitor** — TimedAction that marks applications stuck in `REVIEWING_DOCS` for more than 10 minutes as `BLOCKED`.
- **ApplicationEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the canned application fixtures the simulator drips, or swap the agency endpoint URL for a real integration.
- `SPEC.md §5` — adjust the `Application` record fields (e.g., add `urgencyFlag`).
- `prompts/renewal-coordinator.md` — narrow the coordinator's decision scope (e.g., restrict to specific nationality rules).
- `eval-matrix.yaml` — add a `guardrail` control if you want the coordinator's submission proposal checked against an additional policy rule.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a complete renewal request → coordinator plans, documents pass review, caseworker approves, application completes within ~3 minutes.
2. Submit a request with a missing document → workflow halts with `BLOCKED`, reason identifies the specific missing item.
3. Submit a complete request and click **Reject** in the caseworker pane during `AWAITING_CASEWORKER` — application ends in `REJECTED` with the provided reason; the agency endpoint is never called.
4. Verify that a raw passport number in the applicant payload never appears in any LLM prompt — only the PII token is visible to the agents.

## License

Apache 2.0.
