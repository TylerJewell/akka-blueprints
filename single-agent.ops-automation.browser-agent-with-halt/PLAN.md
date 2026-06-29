# PLAN — web-navigation-agent

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[SessionEndpoint]:::ep
  App[AppEndpoint]:::ep
  Session[SessionEntity]:::ese
  Halt[HaltController]:::ese
  Consumer[ScreenshotConsumer]:::cons
  WF[NavigationWorkflow]:::wf
  Agent[WebNavigatorAgent]:::agent
  Guard[ActionGuardrail]:::guard
  Browser[HeadlessBrowserClient]:::guard
  View[SessionView]:::view

  API -->|start session| Session
  API -->|setHalt/clearHalt| Halt
  API -->|approve/reject| WF
  Session -.->|ActionExecuted| Consumer
  Consumer -->|recordScreenshot| Session
  Consumer -->|screenshotBytes| WF
  WF -->|poll halt| Halt
  WF -->|markNavigating / recordAction / requestHitl / complete / halt / fail| Session
  WF -->|actionStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|allow / block / flag-hitl| Agent
  WF -->|navigateTo / click / type / scroll| Browser
  Agent -->|BrowserAction| WF
  Session -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, no HITL)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as SessionEndpoint
  participant S as SessionEntity
  participant H as HaltController
  participant WF as NavigationWorkflow
  participant A as WebNavigatorAgent
  participant G as ActionGuardrail
  participant B as HeadlessBrowserClient
  participant Sc as ScreenshotConsumer

  U->>API: POST /api/sessions
  API->>S: start(goal)
  S-->>API: { sessionId }
  API->>WF: start(sessionId)
  WF->>B: navigateTo(startingUrl)
  WF->>B: captureScreenshot()
  B-->>WF: screenshotBytes
  WF->>S: markNavigating
  loop Each action step (until COMPLETE or maxSteps)
    WF->>H: getHalt()
    H-->>WF: halted=false
    WF->>A: runSingleTask(goal + screenshot attachment)
    A->>G: before-tool-call(BrowserAction)
    G-->>A: allow
    A-->>WF: BrowserAction
    WF->>B: execute action (click/type/scroll/navigate)
    WF->>S: recordAction(outcome)
    S-.->>Sc: ActionExecuted
    Sc->>B: captureScreenshot()
    Sc->>S: recordScreenshot(path)
  end
  WF->>S: complete(outcome)
  S-.->>U: SSE event(COMPLETED)
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> STARTING
  STARTING --> NAVIGATING: SessionStarted / markNavigating
  NAVIGATING --> AWAITING_APPROVAL: HitlRequested (high-stakes action)
  AWAITING_APPROVAL --> NAVIGATING: HitlResolved (APPROVED)
  AWAITING_APPROVAL --> NAVIGATING: HitlResolved (REJECTED, agent replans)
  AWAITING_APPROVAL --> FAILED: HITL timeout
  NAVIGATING --> COMPLETED: SessionCompleted
  NAVIGATING --> HALTED: SessionHalted (halt flag set)
  NAVIGATING --> FAILED: SessionFailed (agent error)
  AWAITING_APPROVAL --> HALTED: SessionHalted (halt flag set)
  COMPLETED --> [*]
  HALTED --> [*]
  FAILED --> [*]
  REJECTED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionStarted : emits
  SessionEntity ||--o{ ActionPlanned : emits
  SessionEntity ||--o{ ActionExecuted : emits
  SessionEntity ||--o{ ScreenshotCaptured : emits
  SessionEntity ||--o{ HitlRequested : emits
  SessionEntity ||--o{ HitlResolved : emits
  SessionEntity ||--o{ SessionCompleted : emits
  SessionEntity ||--o{ SessionHalted : emits
  SessionEntity ||--o{ SessionFailed : emits
  HaltController ||--o{ HaltSet : emits
  HaltController ||--o{ HaltCleared : emits
  SessionView }o--|| SessionEntity : projects
  ScreenshotConsumer }o--|| SessionEntity : subscribes
  NavigationWorkflow }o--|| SessionEntity : reads-and-writes
  NavigationWorkflow }o--|| HaltController : reads
  WebNavigatorAgent ||--o{ BrowserAction : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SessionEndpoint` | `api/SessionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `HaltController` | `application/HaltController.java` (state in `domain/HaltState.java`, events in `domain/HaltEvent.java`) |
| `ScreenshotConsumer` | `application/ScreenshotConsumer.java` |
| `NavigationWorkflow` | `application/NavigationWorkflow.java` |
| `WebNavigatorAgent` | `application/WebNavigatorAgent.java` (tasks in `application/NavigationTasks.java`) |
| `ActionGuardrail` | `application/ActionGuardrail.java` |
| `HeadlessBrowserClient` | `application/HeadlessBrowserClient.java` |
| `SessionView` | `application/SessionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `initStep` 30 s, `actionStep` 90 s, `hitlStep` 300 s, `completeStep` 10 s, `haltedStep` 5 s, `failedStep` 5 s. Default step recovery `maxRetries(2).failoverTo(NavigationWorkflow::failedStep)`. The 90 s on `actionStep` accommodates LLM vision latency plus browser execution time (Lesson 4).
- **Halt semantics**: the `HaltController` entity is global (single id `"global"`). Any concurrent session's workflow reads it before each step. The halt flag is set with `HaltSet` and cleared with `HaltCleared`; the entity persists across restarts so a crash-and-restart does not inadvertently un-halt an active operator halt.
- **HITL resume**: the workflow's `hitlStep` suspends until `SessionEndpoint` calls `NavigationWorkflow.resume(HitlDecision)`. No polling; the workflow is woken by the external signal. If the timeout expires first, the workflow's default step recovery fires and transitions to `failedStep`.
- **One agent per session**: the AutonomousAgent instance id is `"navigator-" + sessionId`, giving each navigation its own conversation context. `maxIterationsPerTask(5)` caps guardrail-triggered retries per action step.
- **Screenshot loop**: the `ScreenshotConsumer` is at-least-once; it checks the latest `ActionOutcome.screenshotPath` before re-capturing to avoid duplicate screenshots on redelivery.
- **No saga / no compensation**: browser actions are not transactional. A `CLICK` that submits a form cannot be rolled back. This is precisely why HITL approval exists for high-stakes actions — the workflow requires explicit human confirmation before any irreversible action executes.
