# PLAN — akka-cx-with-escalation

Architectural sketch for OpenAI Agents Customer Service with Escalation. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1a1a2e", "primaryTextColor": "#ffffff", "primaryBorderColor": "#7EC8E3", "lineColor": "#cccccc", "secondaryColor": "#16213e", "tertiaryColor": "#0f3460"}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[SupportEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[SupportWorkflow]:::wf
  SA[SupportAgent]:::agent
  EA[EscalationAgent]:::agent
  CE[ConversationEntity]:::ese
  CV[ConversationsView]:::view

  EP -->|new conversation| WF
  WF -->|reply task| SA
  WF -->|handoff task| EA
  SA -->|recordReply| CE
  EA -->|recordHandoff| CE
  EP -->|escalate / accept / resolve| CE
  WF -->|poll status| CE
  CE -.->|events| CV
  EP -->|getAllConversations / SSE| CV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1a1a2e", "primaryTextColor": "#ffffff", "primaryBorderColor": "#7EC8E3", "lineColor": "#cccccc", "secondaryColor": "#16213e"}}}%%
sequenceDiagram
  autonumber
  actor Customer
  actor HumanAgent as Human Agent
  participant EP as SupportEndpoint
  participant WF as SupportWorkflow
  participant SA as SupportAgent
  participant CE as ConversationEntity
  participant EA as EscalationAgent

  Customer->>EP: POST /api/conversations {customerMessage}
  EP->>WF: start(conversationId, customerMessage)
  WF->>SA: runSingleTask(REPLY)
  SA-->>WF: AgentReply{message, action}
  WF->>CE: recordReply -> ACTIVE
  Note over WF,CE: awaitEscalationDecisionStep polls every 5s
  Customer->>EP: POST /api/conversations/{id}/escalate
  EP->>CE: requestEscalation -> ESCALATING
  WF->>CE: getConversation -> ESCALATING
  HumanAgent->>EP: POST /api/conversations/{id}/accept-escalation
  EP->>CE: acceptEscalation -> ESCALATING (acceptedBy set)
  WF->>EA: runSingleTask(HANDOFF) [guard: status == ESCALATING + acceptedBy present]
  EA-->>WF: HandoffSummary{summary, priority}
  WF->>CE: recordHandoff -> ESCALATED
```

## State machine

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1a1a2e", "primaryTextColor": "#ffffff", "primaryBorderColor": "#7EC8E3", "lineColor": "#cccccc", "secondaryColor": "#16213e", "transitionLabelColor": "#cccccc"}}}%%
stateDiagram-v2
  [*] --> ACTIVE: ConversationOpened
  ACTIVE --> ESCALATING: EscalationRequested
  ACTIVE --> RESOLVED: ConversationResolved
  ESCALATING --> ESCALATED: EscalationAccepted + recordHandoff
  ESCALATED --> [*]
  RESOLVED --> [*]
```

## Entity model

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1a1a2e", "primaryTextColor": "#ffffff", "primaryBorderColor": "#7EC8E3", "lineColor": "#cccccc"}}}%%
erDiagram
  CONVERSATION ||--o{ CONVERSATION_EVENT : emits
  CONVERSATION {
    string id
    string customerMessage
    string status
    string agentReply
    string escalationReason
    string handoffSummary
    string handoffPriority
  }
  CONVERSATION_EVENT {
    string type
    string occurredAt
  }
  CONVERSATIONS_VIEW {
    string id
    string status
    string agentReply
  }
  CONVERSATION ||--|| CONVERSATIONS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| SupportAgent | `application/SupportAgent.java` |
| EscalationAgent | `application/EscalationAgent.java` |
| SupportWorkflow | `application/SupportWorkflow.java` |
| SupportTasks | `application/SupportTasks.java` |
| ConversationEntity | `application/ConversationEntity.java` |
| ConversationsView | `application/ConversationsView.java` |
| SupportEndpoint | `api/SupportEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Conversation / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `replyStep` and `acceptEscalationStep` call agents; both set `stepTimeout(60s)` to absorb LLM latency. The default 5 s step timeout would retry prematurely (Lesson 4).
- **Await-escalation-decision task.** The workflow does not block a thread; `awaitEscalationDecisionStep` reads `ConversationEntity.getConversation`, and on `ACTIVE` it self-schedules a 5-second resume timer until the conversation moves to `ESCALATING` or `RESOLVED`.
- **Escalation acceptance poll.** `acceptEscalationStep` similarly polls until `acceptedBy` is present on the entity before calling `EscalationAgent`.
- **Idempotency.** `conversationId` is the workflow id and entity id; re-delivery of `recordReply` / `recordHandoff` is absorbed by event-applier status checks.
- **Escalation guard.** Before the handoff task runs, the workflow verifies `ConversationEntity.status == ESCALATING` and `acceptedBy` is non-empty. No compensation is needed because the handoff is the terminal write.
