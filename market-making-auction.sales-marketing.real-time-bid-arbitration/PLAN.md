# PLAN — Real-Time Bid Arbitration Agent

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  BE[BidEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  ARQ[AuctionRequestQueue<br/>EventSourcedEntity]:::ese
  ARC[AuctionRequestConsumer<br/>Consumer]:::con
  WF[BidWorkflow<br/>Workflow]:::wf
  BAA[BidArbitrationAgent<br/>AutonomousAgent]:::ag
  ASE[AuctionSlotEntity<br/>EventSourcedEntity]:::ese
  CBE[CampaignBudgetEntity<br/>EventSourcedEntity]:::ese
  VW[BidLedgerView<br/>View]:::vw
  SIM[AuctionSimulator<br/>TimedAction]:::ta
  PS[PerformanceSampler<br/>TimedAction]:::ta

  BE -->|POST /bids| ARQ
  SIM -.->|every 30s| ARQ
  ARQ -.->|AuctionSlotSubmitted| ARC
  ARC -->|start workflow| WF
  WF -->|getBudget| CBE
  WF -->|EVALUATE_IMPRESSION| BAA
  WF -->|commands| ASE
  WF -->|chargeBid| CBE
  ASE -.->|events| VW
  CBE -.->|events| VW
  PS -.->|every 5m| VW
  PS -->|recordPerformanceEval| ASE
  BE -->|getAllSlots / SSE| VW
  AE --> STATIC[static-resources]:::static

  classDef ep fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ese fill:#141414,stroke:#F5C518,color:#fff;
  classDef vw fill:#141414,stroke:#3fb950,color:#fff;
  classDef wf fill:#141414,stroke:#ff5f57,color:#fff;
  classDef ag fill:#141414,stroke:#B388FF,color:#fff;
  classDef con fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ta fill:#141414,stroke:#F5C518,color:#fff;
  classDef static fill:#0A0A0A,stroke:#333,color:#aaa;
```

Solid arrows: synchronous commands. Dashed arrows: event subscriptions or scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant BE as BidEndpoint
  participant ARQ as AuctionRequestQueue
  participant WF as BidWorkflow
  participant CBE as CampaignBudgetEntity
  participant BAA as BidArbitrationAgent
  participant ASE as AuctionSlotEntity

  U->>BE: POST /api/bids {slotId, campaignId, ...}
  BE->>ARQ: submitSlot
  ARQ-->>WF: AuctionRequestConsumer starts workflow
  WF->>ASE: receiveSlot (RECEIVED)
  WF->>CBE: getBudget
  alt cap breached
    WF->>ASE: haltSlot (HALTED)
  else cap OK
    WF->>ASE: evaluateSlot (EVALUATING)
    WF->>BAA: EVALUATE_IMPRESSION -> BidDecision
    alt shouldBid = false
      WF->>ASE: skipSlot (SKIPPED)
    else shouldBid = true
      WF->>ASE: placeBid (BID_PLACED)
      Note over WF: burst check; if > threshold -> escalateStep
      alt burst detected
        WF->>ASE: escalateSlot (ESCALATED)
      else no burst
        WF->>ASE: recordResult (WON or LOST)
        WF->>CBE: chargeBid
      end
    end
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> EVALUATING: budget OK
  RECEIVED --> HALTED: cap breached
  EVALUATING --> SKIPPED: shouldBid = false
  EVALUATING --> BID_PLACED: bid submitted
  BID_PLACED --> ESCALATED: burst threshold exceeded
  BID_PLACED --> WON: exchange win
  BID_PLACED --> LOST: exchange loss
  HALTED --> [*]
  SKIPPED --> [*]
  ESCALATED --> [*]
  WON --> WON: PerformanceEvalRecorded
  LOST --> LOST: PerformanceEvalRecorded
  WON --> [*]
  LOST --> [*]
```

## Entity model

```mermaid
erDiagram
  AUCTION_SLOT ||--o| BID_DECISION : has
  AUCTION_SLOT ||--o| BID_RESULT : has
  AUCTION_SLOT ||--o| PERFORMANCE_SNAPSHOT : has
  AUCTION_REQUEST_QUEUE ||--|| AUCTION_SLOT : seeds
  CAMPAIGN_BUDGET ||--o{ AUCTION_SLOT : governs
  AUCTION_SLOT {
    string slotId
    string campaignId
    enum status
    decimal floorPrice
    string placementId
    instant receivedAt
  }
  CAMPAIGN_BUDGET {
    string campaignId
    decimal windowCapAmount
    decimal windowSpent
    boolean capBreached
    instant windowStart
  }
  AUCTION_REQUEST_QUEUE {
    string slotId
    string campaignId
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `BidArbitrationAgent` | AutonomousAgent | `application/BidArbitrationAgent.java` |
| `BidTasks` | Task constants | `application/BidTasks.java` |
| `BidWorkflow` | Workflow | `application/BidWorkflow.java` |
| `AuctionSlotEntity` | EventSourcedEntity | `domain/AuctionSlotEntity.java` |
| `CampaignBudgetEntity` | EventSourcedEntity | `domain/CampaignBudgetEntity.java` |
| `AuctionRequestQueue` | EventSourcedEntity | `domain/AuctionRequestQueue.java` |
| `BidLedgerView` | View | `application/BidLedgerView.java` |
| `AuctionRequestConsumer` | Consumer | `application/AuctionRequestConsumer.java` |
| `AuctionSimulator` | TimedAction | `application/AuctionSimulator.java` |
| `PerformanceSampler` | TimedAction | `application/PerformanceSampler.java` |
| `BidEndpoint` | HttpEndpoint | `api/BidEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeout (Lesson 4):** `evaluateStep` (the agent call) gets 20 s; the 5 s default fails LLM calls. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Idempotency:** the workflow id is the `slotId`. Re-delivery of the same `AuctionSlotSubmitted` event resolves to the same workflow instance — no duplicate slot.
- **Halt path:** `budgetCheckStep` reads `CampaignBudgetEntity.getBudget` synchronously before calling the agent. If `capBreached` is true, the workflow transitions to `haltStep` without ever invoking `BidArbitrationAgent`.
- **Burst detection:** an in-memory counter keyed by `campaignId + minute-bucket` increments on each successful `placeStep`. If the count exceeds `burstThreshold`, the workflow transitions to `escalateStep`. The counter resets per window reset.
- **Performance eval:** `PerformanceSampler` reads `BidLedgerView.getAllSlots` (no enum WHERE clause — Lesson 2) and filters client-side for slots resolved in the current window.
