# PLAN — slack-lead-qualifier

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Poller[SlackEventPoller]:::ta
  Queue[SlackEventQueue]:::ese
  Enricher[LeadEnrichmentAgent]:::agent
  Scorer[LeadScoringAgent]:::agent
  Sanitizer[PiiSanitizer]:::cons
  WF[LeadWorkflow]:::wf
  Entity[LeadEntity]:::ese
  View[LeadView]:::view
  API[LeadEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 20s| Queue
  Queue -.->|MemberJoinedReceived| WF
  WF -->|call| Enricher
  WF -->|emit LeadEnriched| Entity
  Entity -.->|LeadEnriched| Sanitizer
  Sanitizer -->|emit LeadSanitized| Entity
  WF -->|poll until SANITIZED| Entity
  WF -->|call| Scorer
  WF -->|postToSlack tool + guardrail| Entity
  Entity -.->|projects| View
  API -->|query/SSE| View
  API -->|get lead| Entity
```

## Interaction sequence — J1 (happy path: HOT lead posted)

```mermaid
sequenceDiagram
  autonumber
  participant P as SlackEventPoller
  participant Q as SlackEventQueue
  participant W as LeadWorkflow
  participant E as LeadEnrichmentAgent
  participant En as LeadEntity
  participant S as PiiSanitizer
  participant Sc as LeadScoringAgent
  participant G as SlackPostGuardrail
  participant Sl as postToSlack (stub)

  P->>Q: emit MemberJoinedReceived
  Q->>W: start({leadId, event})
  W->>E: enrich(userId, displayName, channelId)
  E-->>W: EnrichmentResult
  W->>En: emit LeadEnriched
  En->>S: LeadEnriched (Consumer)
  S->>En: emit LeadSanitized
  W->>En: poll until status == SANITIZED
  En-->>W: SANITIZED
  W->>Sc: score(sanitizedEnrichment)
  Sc-->>W: ScoringResult{score=82, tier=HOT}
  W->>En: emit LeadScored + PostDrafted
  W->>G: before-tool-call check(draft)
  G-->>W: approved
  W->>Sl: postToSlack(channel, body)
  Sl-->>W: slackTimestamp
  W->>En: emit LeadPosted
```

## State machine — `LeadEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> ENRICHED: LeadEnriched
  ENRICHED --> SANITIZED: LeadSanitized
  ENRICHED --> FAILED: enrichment sentinel / sanitizer error
  SANITIZED --> SCORED: LeadScored
  SCORED --> POSTING: PostDrafted (score >= threshold)
  SCORED --> SUPPRESSED: score below threshold
  POSTING --> POSTED: LeadPosted (guardrail pass)
  POSTING --> FAILED: LeadFailed (guardrail violation)
  POSTED --> [*]
  SUPPRESSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  LeadEntity ||--o{ MemberJoinedReceived : emits
  LeadEntity ||--o{ LeadEnriched : emits
  LeadEntity ||--o{ LeadSanitized : emits
  LeadEntity ||--o{ LeadScored : emits
  LeadEntity ||--o{ PostDrafted : emits
  LeadEntity ||--o{ LeadPosted : emits
  LeadEntity ||--o{ LeadSuppressed : emits
  LeadEntity ||--o{ LeadFailed : emits
  LeadView }o--|| LeadEntity : projects
  SlackEventQueue ||--o{ MemberJoinedReceived : emits
  PiiSanitizer }o--|| LeadEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SlackEventPoller` | `application/SlackEventPoller.java` |
| `SlackEventQueue` | `application/SlackEventQueue.java` |
| `LeadEnrichmentAgent` | `application/LeadEnrichmentAgent.java` |
| `LeadScoringAgent` | `application/LeadScoringAgent.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `LeadWorkflow` | `application/LeadWorkflow.java` |
| `LeadEntity` | `application/LeadEntity.java` (state in `domain/Lead.java`, events in `domain/LeadEvent.java`) |
| `LeadView` | `application/LeadView.java` |
| `LeadEndpoint` | `api/LeadEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: enrichment 30 s, scoring 20 s. On enrichment timeout, set EnrichmentResult to sentinel and continue; PiiSanitizer detects the sentinel and emits LeadFailed.
- **Sanitizer poll**: `waitSanitizedStep` polls `LeadEntity.getLead` every 3 s. No auto-timeout — if the Consumer is delayed, the workflow waits. Deployers wanting an SLA can add a TimedAction that marks stale leads FAILED after N minutes.
- **Guardrail placement**: the `SlackPostGuardrail` runs before every `postToSlack` tool invocation. It reads the draft body from the call argument — it does NOT re-read entity state.
- **Idempotency**: each workflow uses `leadId` (derived from `eventId`) as the workflow id so duplicate event deliveries fold into one workflow.
- **Threshold configuration**: `akka.javasdk.lead-qualifier.posting-threshold` (default 60) is read at `conditionalPostStep` time; changing the config and restarting takes effect immediately without code changes.
