# API contract — lead-qualifier

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/leads` | `SubmitLeadRequest` | `201 { leadId }` | `LeadEndpoint` → `LeadEntity` |
| `GET` | `/api/leads` | — | `200 [ LeadRow... ]` (newest-first) | `LeadEndpoint` ← `LeadView` |
| `GET` | `/api/leads/{id}` | — | `200 LeadRow` / `404` | `LeadEndpoint` ← `LeadView` |
| `GET` | `/api/leads/sse` | — | `text/event-stream` | `LeadEndpoint` ← `LeadView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitLeadRequest (request body)

```json
{
  "rawInquiry": "Enterprise SaaS pricing for 500 seats"
}
```

### LeadRow (response body)

```json
{
  "leadId": "lead-5af9c2...",
  "rawInquiry": "Enterprise SaaS pricing for 500 seats",
  "form": {
    "contact": {
      "nameInitial": "J.D.",
      "company": "Acme Corp",
      "channel": "web-form"
    },
    "intent": {
      "productInterest": "Enterprise SaaS",
      "useCaseSummary": "Pricing inquiry for 500-seat deployment",
      "budgetIndicator": "50k+"
    },
    "capturedAt": "2026-06-28T10:00:00Z"
  },
  "score": {
    "fit": { "score": 85, "reasoning": "Large-seat count, high budget indicator" },
    "urgency": "HIGH",
    "recommendedStage": "SQL",
    "disqualificationReason": null,
    "scoredAt": "2026-06-28T10:00:05Z"
  },
  "crmEntry": {
    "leadId": "lead-5af9c2...",
    "stage": "SQL",
    "ownerCandidate": "owner-02",
    "notes": "High-fit enterprise lead. Route to enterprise sales team.",
    "maskedEmail": "a3f9b2c1@masked",
    "maskedPhone": "[REDACTED]",
    "nameInitial": "J.D.",
    "enrichedAt": "2026-06-28T10:00:10Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Stage valid, owner assigned, notes present, fit-score consistent with stage.",
    "evaluatedAt": "2026-06-28T10:00:10Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:10Z",
  "guardrailRejections": []
}
```

Notes on the response shape:

- `form.contact` never includes `rawEmail` or `rawPhone`. Those fields are stored on `LeadEntity` and are never projected into `LeadRow` or included in any API response.
- Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).
- `guardrailRejections` is an array — empty on the happy path, populated when the agent attempted a misordered or schema-invalid tool call.

### Guardrail rejection shape (element of `guardrailRejections`)

```json
{
  "phase": "CAPTURE",
  "tool": "writeCrmStage",
  "reason": "phase-violation: writeCrmStage requires status in {QUALIFIED, ENRICHING} with score present, saw CAPTURING",
  "rejectedAt": "2026-06-28T10:00:01Z"
}
```

Schema-violation rejection example:

```json
{
  "phase": "ENRICH",
  "tool": "writeCrmStage",
  "reason": "schema-violation: stage 'Prospect' is not in allowed set {New, Qualified, MQL, SQL, Disqualified}",
  "rejectedAt": "2026-06-28T10:00:08Z"
}
```

### SSE event format

```
event: lead-update
data: { "leadId": "lead-5af9c2...", "status": "QUALIFIED", "score": { ... }, ... }
```

One event per state transition (`CREATED`, `CAPTURING`, `CAPTURED`, `QUALIFYING`, `QUALIFIED`, `ENRICHING`, `ENRICHED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: lead-rejection
data: { "leadId": "lead-5af9c2...", "phase": "CAPTURE", "tool": "writeCrmStage", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `leadId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

SSE events never include `rawEmail` or `rawPhone` — the `LeadRow` projection excludes them. Only `maskedEmail`, `maskedPhone`, and `nameInitial` appear in the CRM entry payload.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitLeadRequest` record and the `LeadCreated` event to carry it.
