# PLAN — Contract-Net Task Auctioneer

## 1. Component Graph

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'primaryBorderColor': '#4a90d9', 'lineColor': '#4a90d9', 'secondaryColor': '#16213e', 'tertiaryColor': '#0f3460', 'edgeLabelBackground': '#16213e', 'clusterBkg': '#16213e', 'titleColor': '#e0e0e0', 'nodeBorder': '#4a90d9', 'mainBkg': '#1a1a2e'}}}%%
graph TD
    subgraph HTTP["HTTP Layer"]
        EP[TaskAuctioneerEndpoint]
    end

    subgraph Entities["Entities"]
        TAE[TaskAnnouncementEntity]
        CRE[ContractorRegistryEntity]
    end

    subgraph Workflows["Workflows"]
        AW[AuctionWorkflow]
        RW[RenegotiationWorkflow]
    end

    subgraph Agents["AI Agents"]
        BEA[BidEvaluatorAgent]
        PSA[PerformanceScorerAgent]
    end

    subgraph Views["Views"]
        AAV[ActiveAuctionsView]
        CLV[ContractorLeaderboardView]
    end

    EP -->|AnnounceTask| TAE
    EP -->|SubmitBid| TAE
    EP -->|RegisterContractor| CRE
    EP -->|SSE stream| AAV
    EP -->|SSE stream| CLV

    TAE -->|TaskAnnounced| AW
    AW -->|rank bids| BEA
    AW -->|AwardTask| TAE
    AW -->|score outcome| PSA
    PSA -->|UpdateScore| CRE
    TAE -->|TaskFailed| RW
    RW -->|reopens bidding| TAE
    CRE -->|ContractorThrottled| AW

    TAE -.->|projects| AAV
    CRE -.->|projects| CLV
```

## 2. Sequence — Happy Path (Announce → Award → Complete → Score)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'primaryBorderColor': '#4a90d9', 'lineColor': '#4a90d9', 'secondaryColor': '#16213e', 'actorBkg': '#0f3460', 'actorBorder': '#4a90d9', 'actorTextColor': '#e0e0e0', 'activationBkgColor': '#16213e', 'activationBorderColor': '#4a90d9', 'noteBkgColor': '#16213e', 'noteTextColor': '#e0e0e0'}}}%%
sequenceDiagram
    participant UI as UI / Caller
    participant EP as TaskAuctioneerEndpoint
    participant TAE as TaskAnnouncementEntity
    participant AW as AuctionWorkflow
    participant BEA as BidEvaluatorAgent
    participant CRE as ContractorRegistryEntity
    participant PSA as PerformanceScorerAgent

    UI->>EP: POST /tasks {spec, deadline}
    EP->>TAE: AnnounceTask
    TAE-->>AW: TaskAnnounced (starts workflow)
    AW-->>AW: start bidding timer

    loop Bidding window open
        UI->>EP: POST /tasks/{id}/bids {contractorId, cost}
        EP->>TAE: SubmitBid
        TAE-->>AW: BidSubmitted
    end

    AW->>BEA: rank bids against task spec
    BEA-->>AW: ranked bid list + rationale

    Note over AW: Guardrail: validate winning bid
    AW->>CRE: check throttle flag + capabilities
    CRE-->>AW: contractor status
    AW->>TAE: AwardTask (if valid)
    TAE-->>AW: TaskAwarded

    UI->>EP: POST /tasks/{id}/complete {report}
    EP->>TAE: ReportCompletion
    TAE-->>AW: TaskCompleted

    AW->>PSA: score completion against task spec
    PSA-->>AW: {score: 87, feedback: "..."}
    AW->>CRE: UpdateScore {contractorId, score}
    CRE-->>AW: PerformanceScored
```

## 3. State Machine — TaskAnnouncementEntity

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'primaryBorderColor': '#4a90d9', 'lineColor': '#4a90d9', 'secondaryColor': '#16213e', 'tertiaryColor': '#0f3460', 'edgeLabelBackground': '#16213e'}}}%%
stateDiagram-v2
    classDef announced fill:#0f3460,stroke:#4a90d9,color:#e0e0e0
    classDef bidding fill:#163460,stroke:#4a90d9,color:#e0e0e0
    classDef awarded fill:#1a4a60,stroke:#4a90d9,color:#e0e0e0
    classDef inprogress fill:#1a5a40,stroke:#4ad9a0,color:#e0e0e0
    classDef terminal fill:#3a1a1a,stroke:#d94a4a,color:#e0e0e0
    classDef reneg fill:#3a2a10,stroke:#d9a04a,color:#e0e0e0

    [*] --> ANNOUNCED: AnnounceTask
    ANNOUNCED --> BIDDING: bidding window opens
    BIDDING --> BIDDING: BidSubmitted
    BIDDING --> AWARDED: AwardTask (guardrail passed)
    AWARDED --> IN_PROGRESS: TaskStarted
    IN_PROGRESS --> COMPLETED: ReportCompletion
    IN_PROGRESS --> FAILED: ReportFailure
    FAILED --> RENEGOTIATING: RenegotiationOpened
    RENEGOTIATING --> BIDDING: renegotiation bidding opens
    COMPLETED --> [*]

    class ANNOUNCED announced
    class BIDDING bidding
    class AWARDED awarded
    class IN_PROGRESS inprogress
    class COMPLETED terminal
    class FAILED terminal
    class RENEGOTIATING reneg
```

## 4. Entity-Relationship Diagram

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'primaryBorderColor': '#4a90d9', 'lineColor': '#4a90d9', 'secondaryColor': '#16213e', 'tertiaryColor': '#0f3460', 'edgeLabelBackground': '#16213e'}}}%%
erDiagram
    TASK_ANNOUNCEMENT {
        string taskId PK
        string title
        string description
        string_list requiredCapabilities
        decimal maxBudget
        int biddingDeadlineSeconds
        string status
        string winningBidId FK
        instant announcedAt
        int renegotiationCount
    }

    BID {
        string bidId PK
        string taskId FK
        string contractorId FK
        decimal costEstimate
        string capacityNote
        instant submittedAt
        boolean guardrailPassed
        string evaluatorRationale
    }

    CONTRACTOR_PROFILE {
        string contractorId PK
        string name
        string_list capabilities
        int performanceScore
        boolean throttled
        int completedTasks
        instant registeredAt
    }

    PERFORMANCE_RECORD {
        string recordId PK
        string contractorId FK
        string taskId FK
        int score
        string feedback
        instant scoredAt
    }

    TASK_ANNOUNCEMENT ||--o{ BID : "receives"
    CONTRACTOR_PROFILE ||--o{ BID : "submits"
    CONTRACTOR_PROFILE ||--o{ PERFORMANCE_RECORD : "accumulates"
    TASK_ANNOUNCEMENT ||--o{ PERFORMANCE_RECORD : "generates"
```
