# Architecture Plan: Modular Agent Skills Loader

## Diagram 1: Component Graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a2e','primaryTextColor':'#e0e0e0','primaryBorderColor':'#4a4a8a','lineColor':'#6a6aaa','secondaryColor':'#16213e','tertiaryColor':'#0f3460'}}}%%
graph TD
    Client([HTTP Client]):::external

    subgraph Endpoint Layer
        AOE[AgentOrchestrationEndpoint<br/>HTTP Endpoint]:::endpoint
    end

    subgraph Agent Layer
        TDA[TaskDispatchAgent<br/>Agent]:::agent
        GL{Guardrail<br/>before-tool-invocation}:::guardrail
    end

    subgraph Workflow Layer
        SLW[SkillLoaderWorkflow<br/>Workflow]:::workflow
    end

    subgraph Entity Layer
        SRE[SkillRegistryEntity<br/>Key-Value Entity]:::entity
    end

    subgraph View Layer
        SEV[SkillExecutionView<br/>View]:::view
    end

    Client -->|POST /tasks<br/>POST /skills<br/>PUT /skills/{id}/enable|disable| AOE
    Client -->|GET /tasks SSE| AOE
    AOE -->|dispatch task| TDA
    AOE -->|register/toggle skill| SRE
    AOE -->|query tasks| SEV
    TDA -->|registry-lookup tool| SRE
    TDA -->|invoke skill tool| GL
    GL -->|approved| SLW
    GL -->|rejected| TDA
    SLW -->|validate capability| SRE
    SLW -->|emit SkillLoaded / SkillRejected| SEV
    SRE -->|events| SEV

    classDef external fill:#0f3460,stroke:#4a4a8a,color:#e0e0e0
    classDef endpoint fill:#16213e,stroke:#4a4a8a,color:#e0e0e0
    classDef agent fill:#1a1a2e,stroke:#6a6aaa,color:#e0e0e0
    classDef guardrail fill:#2a1a3e,stroke:#9a4a9a,color:#e0e0e0
    classDef workflow fill:#1a2e1a,stroke:#4a8a4a,color:#e0e0e0
    classDef entity fill:#2e1a1a,stroke:#8a4a4a,color:#e0e0e0
    classDef view fill:#1a2e2e,stroke:#4a8a8a,color:#e0e0e0
```

---

## Diagram 2: Request Sequence

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a2e','primaryTextColor':'#e0e0e0','primaryBorderColor':'#4a4a8a','lineColor':'#6a6aaa','secondaryColor':'#16213e','tertiaryColor':'#0f3460'}}}%%
sequenceDiagram
    participant C as Client
    participant AOE as AgentOrchestrationEndpoint
    participant TDA as TaskDispatchAgent
    participant GL as Guardrail
    participant SLW as SkillLoaderWorkflow
    participant SRE as SkillRegistryEntity
    participant SEV as SkillExecutionView

    C->>AOE: POST /tasks {description, capability}
    AOE->>TDA: dispatch(taskId, description, capability)
    TDA->>SRE: registry-lookup(capability)
    SRE-->>TDA: [SkillDescriptor, ...]
    TDA->>TDA: select best skill (highest version)
    Note over TDA: status → SKILL_SELECTED

    TDA->>GL: invoke skill tool (skillId, capability)
    GL->>SRE: get(skillId)
    SRE-->>GL: SkillDescriptor

    alt capability present and skill enabled
        GL->>SLW: startLoad(taskId, skillId, capability)
        SLW->>SRE: validate(skillId, capability)
        SRE-->>SLW: OK
        Note over SLW: step 1 validate ✓
        SLW->>SLW: load skill folder
        Note over SLW: step 2 load ✓
        SLW->>SRE: re-verify(skillId, capability)
        SRE-->>SLW: OK
        Note over SLW: step 3 verify ✓
        SLW->>SEV: emit SkillLoaded
        Note over TDA: status → GUARDRAIL_PASSED
        TDA->>TDA: execute skill tool
        Note over TDA: status → EXECUTING → COMPLETED
        TDA-->>AOE: result
        AOE-->>C: 200 {taskId, result}
    else skill disabled or capability absent
        GL->>SEV: emit SkillRejected
        Note over TDA: status → REJECTED
        TDA-->>AOE: rejection reason
        AOE-->>C: 422 {taskId, reason}
    end

    AOE->>C: SSE task-update events throughout
```

---

## Diagram 3: Task State Machine

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a2e','primaryTextColor':'#e0e0e0','primaryBorderColor':'#4a4a8a','lineColor':'#6a6aaa','secondaryColor':'#16213e','tertiaryColor':'#0f3460'}}}%%
stateDiagram-v2
    [*] --> PENDING : task submitted

    PENDING --> SKILL_SELECTED : agent selects skill
    PENDING --> REJECTED : no matching skill found

    SKILL_SELECTED --> GUARDRAIL_PASSED : capability verified in registry
    SKILL_SELECTED --> REJECTED : guardrail blocked\n(disabled or unregistered)

    GUARDRAIL_PASSED --> EXECUTING : skill tool invocation begins

    EXECUTING --> COMPLETED : skill returns result
    EXECUTING --> REJECTED : skill tool error

    COMPLETED --> [*]
    REJECTED --> [*]

    note right of GUARDRAIL_PASSED
        SkillLoaded event emitted
        Workflow step 3 (verify) passed
    end note

    note right of REJECTED
        SkillRejected event emitted
        HTTP 422 returned to caller
    end note

    classDef terminal fill:#2e1a1a,stroke:#8a4a4a,color:#e0e0e0
    classDef active fill:#1a2e1a,stroke:#4a8a4a,color:#e0e0e0
    class COMPLETED,REJECTED terminal
    class EXECUTING,GUARDRAIL_PASSED active
```

---

## Diagram 4: Entity-Relationship

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a2e','primaryTextColor':'#e0e0e0','primaryBorderColor':'#4a4a8a','lineColor':'#6a6aaa','secondaryColor':'#16213e','tertiaryColor':'#0f3460'}}}%%
erDiagram
    SkillDescriptor {
        string skillId PK
        string name
        string version
        list_string capabilities
        boolean enabled
        instant loadedAt
    }

    SkillLoadRequest {
        string taskId PK
        string requestedCapability
        string requestedBy
    }

    TaskRecord {
        string taskId PK
        string skillId FK
        string status
        string capability
        instant dispatchedAt
        instant completedAt
    }

    SkillLoaded {
        string skillId FK
        string taskId FK
        instant loadedAt
        list_string capabilities
    }

    SkillRejected {
        string skillId FK
        string taskId FK
        string reason
        instant rejectedAt
    }

    SkillDescriptor ||--o{ SkillLoaded : "approved for"
    SkillDescriptor ||--o{ SkillRejected : "rejected for"
    SkillLoadRequest ||--|| TaskRecord : "creates"
    SkillLoaded ||--|| TaskRecord : "updates status"
    SkillRejected ||--|| TaskRecord : "updates status"
```
