# PLAN — hitl-pdf-extract

Architectural sketch for HITL PDF Data Extraction. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#0e1e2a', 'primaryTextColor': '#7EC8E3', 'primaryBorderColor': '#7EC8E3', 'lineColor': '#888', 'secondaryColor': '#1c1330', 'tertiaryColor': '#1f1900', 'background': '#111', 'mainBkg': '#0d0d0d', 'nodeBorder': '#444', 'clusterBkg': '#1a1a1a', 'titleColor': '#ffffff', 'edgeLabelBackground': '#111', 'transitionLabelColor': '#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[ExtractionEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[ExtractionWorkflow]:::wf
  EA[ExtractionAgent]:::agent
  RA[RedactionAgent]:::agent
  DE[DocumentEntity]:::ese
  DV[DocumentsView]:::view

  EP -->|extraction-request| WF
  WF -->|extract task| EA
  WF -->|redact task| RA
  WF -->|post step| DE
  EA -->|recordExtraction| DE
  RA -->|recordRedaction| DE
  EP -->|approve / reject| DE
  WF -->|poll status| DE
  DE -.->|events| DV
  EP -->|getAllDocuments / SSE| DV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#0e1e2a', 'primaryTextColor': '#e0e0e0', 'primaryBorderColor': '#444', 'lineColor': '#888', 'background': '#111', 'mainBkg': '#0d0d0d', 'actorBkg': '#1a1a1a', 'actorBorder': '#555', 'actorTextColor': '#e0e0e0', 'noteBkgColor': '#1c1c1c', 'noteTextColor': '#ccc', 'activationBkgColor': '#2a2a2a', 'activationBorderColor': '#888', 'signalColor': '#F5C518', 'signalTextColor': '#e0e0e0', 'labelBoxBkgColor': '#1a1a1a', 'labelBoxBorderColor': '#555', 'labelTextColor': '#e0e0e0', 'loopTextColor': '#e0e0e0', 'sequenceNumberColor': '#000'}}}%%
sequenceDiagram
  autonumber
  actor User
  participant EP as ExtractionEndpoint
  participant WF as ExtractionWorkflow
  participant EA as ExtractionAgent
  participant RA as RedactionAgent
  participant DE as DocumentEntity

  User->>EP: POST /api/extraction-request {documentUrl}
  EP->>WF: start(documentId, documentUrl)
  WF->>EA: runSingleTask(EXTRACT)
  EA-->>WF: ExtractionResult{fields, confidence}
  WF->>DE: recordExtraction -> EXTRACTED
  WF->>RA: runSingleTask(REDACT)
  RA-->>WF: RedactedResult{fields}
  WF->>DE: recordRedaction
  Note over WF,DE: confidence < 0.85: setPendingReview; polls every 5s
  User->>EP: POST /api/documents/{id}/approve
  EP->>DE: approve -> APPROVED
  WF->>DE: getDocument -> APPROVED
  WF->>DE: recordPost [guard: status == APPROVED]
  DE-->>WF: PostReceipt{postTarget, postedAt}
  WF->>DE: recordPost -> POSTED
```

## State machine

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#1f1900', 'primaryTextColor': '#F5C518', 'primaryBorderColor': '#F5C518', 'lineColor': '#888', 'secondaryColor': '#0e2010', 'tertiaryColor': '#0e1e2a', 'background': '#111', 'mainBkg': '#0d0d0d', 'stateBkg': '#1a1a1a', 'stateBorder': '#555', 'labelColor': '#ffffff', 'transitionLabelColor': '#cccccc', 'titleColor': '#ffffff', 'nodeBorder': '#555', 'clusterBkg': '#1a1a1a', 'defaultLinkColor': '#888'}}}%%
stateDiagram-v2
  [*] --> EXTRACTED: DocumentExtracted
  EXTRACTED --> PENDING_REVIEW: DocumentPendingReview (low confidence)
  EXTRACTED --> APPROVED: auto-approve (high confidence)
  PENDING_REVIEW --> APPROVED: DocumentApproved
  PENDING_REVIEW --> REJECTED: DocumentRejected
  APPROVED --> POSTED: DocumentPosted
  REJECTED --> [*]
  POSTED --> [*]
```

## Entity model

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#1f1900', 'primaryTextColor': '#F5C518', 'primaryBorderColor': '#F5C518', 'lineColor': '#888', 'background': '#111', 'mainBkg': '#0d0d0d', 'nodeBorder': '#444', 'clusterBkg': '#1a1a1a', 'titleColor': '#ffffff', 'edgeLabelBackground': '#111'}}}%%
erDiagram
  DOCUMENT ||--o{ DOCUMENT_EVENT : emits
  DOCUMENT {
    string id
    string documentUrl
    string status
    double confidence
    map rawFields
    map redactedFields
    string postTarget
  }
  DOCUMENT_EVENT {
    string type
    string occurredAt
  }
  DOCUMENTS_VIEW {
    string id
    string status
    double confidence
  }
  DOCUMENT ||--|| DOCUMENTS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| ExtractionAgent | `application/ExtractionAgent.java` |
| RedactionAgent | `application/RedactionAgent.java` |
| ExtractionWorkflow | `application/ExtractionWorkflow.java` |
| ExtractionTasks | `application/ExtractionTasks.java` |
| DocumentEntity | `application/DocumentEntity.java` |
| DocumentsView | `application/DocumentsView.java` |
| ExtractionEndpoint | `api/ExtractionEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Document / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `extractStep`, `redactStep`, and `postStep` call agents or perform downstream writes; all set `stepTimeout(60s)` to absorb LLM latency. The default 5 s step timeout would expire before most real extractions complete (Lesson 4).
- **Await-review task.** The workflow does not block a thread; `awaitReviewStep` reads `DocumentEntity.getDocument`, and on `PENDING_REVIEW` self-schedules a 5-second resume timer until the human transitions the status.
- **Confidence gate.** When confidence meets the threshold, `awaitReviewStep` calls `approve` directly on `DocumentEntity`, converting EXTRACTED to APPROVED without human input.
- **Post guard.** Before the simulated post tool runs, the before-tool-call guardrail re-reads `DocumentEntity.status`; if it is not `APPROVED`, the call is blocked. No compensation path is needed because posting is the terminal write.
- **Idempotency.** `documentId` is the workflow id and the entity id; re-delivery of `recordExtraction` / `recordPost` is absorbed by event-applier checks on current status.
