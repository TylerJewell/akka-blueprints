# Plan: Valuation Reviewer

> **Akka theme** — all Mermaid diagrams use `%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1A1F36", "primaryTextColor": "#E0E6F0", "primaryBorderColor": "#3D4A6B", "lineColor": "#6B7A99", "secondaryColor": "#252B45", "tertiaryColor": "#1A1F36", "edgeLabelBackground": "#1A1F36", "clusterBkg": "#252B45", "clusterBorder": "#3D4A6B", "titleColor": "#E0E6F0"}}}%%`
>
> State-machine labels that contain spaces or special characters are wrapped in `["label text"]` per Lesson 24.

---

## 1. Component graph

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1A1F36", "primaryTextColor": "#E0E6F0", "primaryBorderColor": "#3D4A6B", "lineColor": "#6B7A99", "secondaryColor": "#252B45", "tertiaryColor": "#1A1F36", "edgeLabelBackground": "#1A1F36", "clusterBkg": "#252B45", "clusterBorder": "#3D4A6B", "titleColor": "#E0E6F0"}}}%%
graph TD
    Client([Client]) --> EP[ValuationEndpoint]
    EP --> WF[ValuationReviewWorkflow]
    WF --> DA[ValuationDrafterAction]
    WF --> CA[ValuationCriticAction]
    DA --> LLM[(LLM)]
    CA --> LLM
    WF --> VW[ValuationReportView]
    EP --> VW
    VW --> SSE([SSE Client])
```

---

## 2. Sequence — happy-path iteration

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1A1F36", "primaryTextColor": "#E0E6F0", "primaryBorderColor": "#3D4A6B", "lineColor": "#6B7A99", "secondaryColor": "#252B45", "tertiaryColor": "#1A1F36", "edgeLabelBackground": "#1A1F36", "clusterBkg": "#252B45", "clusterBorder": "#3D4A6B", "titleColor": "#E0E6F0"}}}%%
sequenceDiagram
    participant C as Client
    participant EP as ValuationEndpoint
    participant WF as ValuationReviewWorkflow
    participant DA as ValuationDrafterAction
    participant CA as ValuationCriticAction
    participant VW as ValuationReportView

    C->>EP: POST /valuations
    EP->>WF: CreateValuationCommand
    WF->>DA: draft(assetDescription, comparables)
    DA-->>WF: ReportDrafted
    WF->>CA: critique(report, reviewStandards)
    CA-->>WF: CritiqueCompleted (some checks fail)
    WF->>DA: revise(report, failingChecks)
    DA-->>WF: ReportRevised
    WF->>CA: critique(revised report, reviewStandards)
    CA-->>WF: CritiqueCompleted (all pass)
    WF-->>VW: SignOffRequested
    C->>EP: POST /valuations/{id}/sign-off
    EP->>WF: SignOffValuationCommand
    WF-->>VW: ValuationSignedOff
    VW-->>C: SSE valuation-updated (SIGNED_OFF)
```

---

## 3. State machine — `ValuationReviewWorkflow`

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1A1F36", "primaryTextColor": "#E0E6F0", "primaryBorderColor": "#3D4A6B", "lineColor": "#6B7A99", "secondaryColor": "#252B45", "tertiaryColor": "#1A1F36", "edgeLabelBackground": "#1A1F36", "clusterBkg": "#252B45", "clusterBorder": "#3D4A6B", "titleColor": "#E0E6F0"}}}%%
stateDiagram-v2
    [*] --> PENDING
    PENDING --> DRAFTING : ["CreateValuationCommand received"]
    DRAFTING --> CRITIQUING : ["ReportDrafted"]
    CRITIQUING --> REVISING : ["checks below threshold"]
    CRITIQUING --> AWAITING_SIGNOFF : ["all checks pass"]
    REVISING --> CRITIQUING : ["ReportRevised"]
    REVISING --> ITERATION_CAP_REACHED : ["cap exhausted"]
    AWAITING_SIGNOFF --> SIGNED_OFF : ["SignOffValuationCommand"]
    AWAITING_SIGNOFF --> REJECTED : ["RejectValuationCommand"]
    ITERATION_CAP_REACHED --> AWAITING_SIGNOFF : ["escalate to reviewer"]
    PENDING --> CANCELLED : ["CancelValuationCommand"]
    DRAFTING --> CANCELLED : ["CancelValuationCommand"]
    CRITIQUING --> CANCELLED : ["CancelValuationCommand"]
    REVISING --> CANCELLED : ["CancelValuationCommand"]
    AWAITING_SIGNOFF --> CANCELLED : ["CancelValuationCommand"]
    SIGNED_OFF --> [*]
    REJECTED --> [*]
    CANCELLED --> [*]
    ITERATION_CAP_REACHED --> [*]
```

---

## 4. Entity–Relationship

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1A1F36", "primaryTextColor": "#E0E6F0", "primaryBorderColor": "#3D4A6B", "lineColor": "#6B7A99", "secondaryColor": "#252B45", "tertiaryColor": "#1A1F36", "edgeLabelBackground": "#1A1F36", "clusterBkg": "#252B45", "clusterBorder": "#3D4A6B", "titleColor": "#E0E6F0"}}}%%
erDiagram
    ValuationReviewWorkflow ||--o{ CritiqueResult : "holds per iteration"
    ValuationReviewWorkflow ||--|| ValuationReportRow : "projected to"
    ValuationReportRow ||--o{ CritiqueResult : "last-iteration results"
    CritiqueResult {
        double compAlignment
        double methodology
        double supportability
        double disclosure
        boolean allPassed
    }
    ValuationReportRow {
        string valuationId
        string assetDescription
        string status
        string currentReport
        int iterationCount
        boolean allPassed
        string reviewerNotes
        timestamp createdAt
        timestamp updatedAt
    }
```
