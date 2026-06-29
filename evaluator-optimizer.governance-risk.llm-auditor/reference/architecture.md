# Architecture — llm-auditor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`AuditEndpoint` is the entry point. It writes a `ResponseReceived` event to `ResponseQueue` (event-sourced for compliance record-keeping). A `Consumer` subscribes to that queue and starts an `AuditWorkflow` per inbound response. The workflow runs a deterministic severity guardrail, then alternates two agents — `CriticAgent` audits, `ReviserAgent` rewrites — until the critic passes the response or the revision ceiling is reached. Each step emits an event on `SessionEntity`. `SessionsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `ResponseSimulator` drips a sample chatbot response every 60 seconds so the UI is not empty when first loaded; `AuditSampler` ticks every 30 seconds and records an `AuditRecorded` event for any completed audit cycle that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first audit returns `REVISE`, the reviser produces a cleaner response, and the second audit passes. The guardrail check, the `CriticAgent` call, the `ReviserAgent` call, and the return to the critic on the revised text are all explicit. Each cycle's events are written to `SessionEntity` before the next step begins so the UI's per-cycle timeline reconstructs the loop without a separate fetch.

## State machine

The session moves through two transient states (`AUDITING`, `REVISING`) and two terminal states (`APPROVED`, `ESCALATED`). The transition from `AUDITING` directly to `ESCALATED` represents the guardrail-blocked path: a response above the severity threshold bypasses the revision loop entirely. The `REVISING → AUDITING` transition is the normal revision path — the workflow calls the critic again on the revised text. `ESCALATED` is also reached when the revision ceiling is hit; both terminal states preserve every cycle's revision and every finding set on the entity.

## Entity model

`SessionEntity` is the system's source of truth; every transition writes one of eight event types. `ResponseQueue` is the compliance log of inbound responses — it is append-only and survives workflow failures. `SessionsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `auditStep` and `reviseStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxRevisions` (default 3). A wall-clock guard could be added by converting `maxRevisions` to a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(escalateStep)` — any unrecoverable agent failure ends in `ESCALATED`, not in a hung workflow.
- Idempotency: `AuditEndpoint.submit` deduplicates on a hash of `(text, channel)` over a 10 s window. `AuditSampler` deduplicates on `(sessionId, cycleNumber)` so a tick that fires twice for the same cycle is a no-op.
- Per-cycle accounting: every cycle is appended to `Session.cycles` with its own input response, guardrail verdict, audit verdict, and (once produced) revised response. The `cycleNumber` is monotonic across the whole loop.
