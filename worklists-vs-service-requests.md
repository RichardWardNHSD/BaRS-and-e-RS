# Worklists vs ServiceRequests: e-RS and BaRS Comparison

## Purpose

This document compares two different approaches to retrieving referral information:

- **e-RS** `POST /STU3/ReferralRequest/$ers.fetchworklist` — the legacy worklist-based approach
- **BaRS** `GET /ServiceRequest` — the standard FHIR R4 approach

The recommendation is that the strategic direction should use BaRS `GET /ServiceRequest` rather than the e-RS worklist concept, with identified changes needed to support the provider use case.

---

## The Two Approaches

### e-RS: Worklist-Based Retrieval

The e-RS API provides a `$ers.fetchworklist` custom FHIR operation that returns referrals as an organisation-scoped worklist:

```
POST /STU3/ReferralRequest/$ers.fetchworklist
```

**Key characteristics:**

| Aspect | Detail |
|---|---|
| **FHIR version** | STU3 |
| **HTTP method** | POST (custom operation, not a RESTful search) |
| **Scope** | Organisation — returns all referrals for a service/organisation |
| **Authentication** | CIS2 (user-restricted) or app-restricted (limited) |
| **Query model** | Parameters resource in the POST body specifying list type, service, filters |
| **Worklist types** | `REFERRALS_FOR_REVIEW`, `REFERRALS_FOR_TRIAGE`, `ADVICE_AND_GUIDANCE_REQUESTS`, etc. |
| **Result** | A list of ReferralRequest resources filtered by organisation, status, and type |
| **User context** | Tied to a specific user's role and organisation (NHSD-eRS-Business-Function header) |

**Example request (e-RS):**

```http
POST /STU3/ReferralRequest/$ers.fetchworklist HTTP/1.1
Host: api.service.nhs.uk
Content-Type: application/fhir+json
NHSD-End-User-Organisation-ODS: R69
NHSD-eRS-Business-Function: SERVICE_PROVIDER_CLINICIAN

{
  "resourceType": "Parameters",
  "meta": {
    "profile": ["https://fhir.nhs.uk/STU3/StructureDefinition/eRS-FetchWorklist-Parameters-1"]
  },
  "parameter": [
    {
      "name": "listType",
      "valueCodeableConcept": {
        "coding": [{
          "system": "https://fhir.nhs.uk/STU3/CodeSystem/eRS-ReferralListSelector-1",
          "code": "REFERRALS_FOR_REVIEW"
        }]
      }
    }
  ]
}
```

### BaRS: ServiceRequest-Based Retrieval

The BaRS API provides a standard FHIR R4 RESTful search on ServiceRequest:

```
GET /ServiceRequest
```

**Key characteristics:**

| Aspect | Detail |
|---|---|
| **FHIR version** | R4 |
| **HTTP method** | GET (standard RESTful search) |
| **Scope** | Patient — returns ServiceRequests for a specific patient |
| **Authentication** | App-restricted (signed JWT) |
| **Query model** | Standard FHIR search parameters in the URL |
| **Search parameters** | `_id`, `patient:identifier` (NHS Number) |
| **Result** | A Bundle of ServiceRequest resources for the specified patient |
| **User context** | System-to-system (no individual user identity required) |

**Example request (BaRS):**

```http
GET /ServiceRequest?patient:identifier=https://fhir.nhs.uk/Id/nhs-number|9876543210 HTTP/1.1
Host: int.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOi...
X-Request-Id: {uuid}
X-Correlation-Id: {uuid}
NHSD-End-User-Organisation: {base64-encoded org}
NHSD-Target-Identifier: {base64-encoded target service}
```

**Example response (BaRS) — patient-scoped:**

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 2,
  "entry": [
    {
      "fullUrl": "https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/ServiceRequest/79120f41-a431-4f08-bcc5-1e67006fcae0",
      "resource": {
        "resourceType": "ServiceRequest",
        "id": "79120f41-a431-4f08-bcc5-1e67006fcae0",
        "meta": {
          "lastUpdated": "2025-06-01T09:30:00+00:00",
          "profile": ["https://fhir.hl7.org.uk/StructureDefinition/UKCore-ServiceRequest"]
        },
        "status": "active",
        "intent": "order",
        "priority": "routine",
        "code": {
          "coding": [{
            "system": "https://fhir.nhs.uk/CodeSystem/usecases-categories-bars",
            "code": "a1t1",
            "display": "111 to ED"
          }]
        },
        "subject": {
          "reference": "Patient/788660eb-d2c9-4773-abd4-318484673fb2",
          "identifier": {
            "system": "https://fhir.nhs.uk/Id/nhs-number",
            "value": "9876543210"
          },
          "display": "John Smith"
        },
        "authoredOn": "2025-06-01T09:15:00+00:00",
        "requester": {
          "reference": "Organization/sender-org-001",
          "identifier": {
            "system": "https://fhir.nhs.uk/Id/ods-organization-code",
            "value": "RYG"
          },
          "display": "Sender GP Practice"
        },
        "performer": [{
          "reference": "Organization/receiver-org-001",
          "identifier": {
            "system": "https://fhir.nhs.uk/Id/ods-organization-code",
            "value": "RXF"
          },
          "display": "Anytown UTC"
        }]
      }
    },
    {
      "fullUrl": "https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/ServiceRequest/a5b6c7d8-e9f0-1234-5678-90abcdef1234",
      "resource": {
        "resourceType": "ServiceRequest",
        "id": "a5b6c7d8-e9f0-1234-5678-90abcdef1234",
        "meta": {
          "lastUpdated": "2025-05-28T14:00:00+00:00",
          "profile": ["https://fhir.hl7.org.uk/StructureDefinition/UKCore-ServiceRequest"]
        },
        "status": "completed",
        "intent": "order",
        "priority": "urgent",
        "code": {
          "coding": [{
            "system": "https://fhir.nhs.uk/CodeSystem/usecases-categories-bars",
            "code": "a1t1",
            "display": "111 to ED"
          }]
        },
        "subject": {
          "reference": "Patient/788660eb-d2c9-4773-abd4-318484673fb2",
          "identifier": {
            "system": "https://fhir.nhs.uk/Id/nhs-number",
            "value": "9876543210"
          },
          "display": "John Smith"
        },
        "authoredOn": "2025-05-28T13:45:00+00:00",
        "requester": {
          "reference": "Organization/sender-org-002",
          "identifier": {
            "system": "https://fhir.nhs.uk/Id/ods-organization-code",
            "value": "A1001"
          },
          "display": "Another GP Practice"
        },
        "performer": [{
          "reference": "Organization/receiver-org-001",
          "identifier": {
            "system": "https://fhir.nhs.uk/Id/ods-organization-code",
            "value": "RXF"
          },
          "display": "Anytown UTC"
        }]
      }
    }
  ]
}
```

**Example response (BaRS) — organisation-scoped (target state):**

```http
GET /ServiceRequest?performer:identifier=https://fhir.nhs.uk/Id/ods-organization-code|RXF&status=active HTTP/1.1
```

Response is the same Bundle format as above, but filtered by the performing organisation rather than patient. The same ServiceRequest resources are returned — the only difference is the query axis.

**Example response (e-RS worklist — for comparison):**

```json
{
  "resourceType": "List",
  "meta": {
    "profile": ["https://fhir.nhs.uk/STU3/StructureDefinition/eRS-Worklist-List-1"]
  },
  "status": "current",
  "mode": "snapshot",
  "entry": [
    {
      "item": {
        "reference": "ReferralRequest/000000070000",
        "display": "UBRN 000000070000"
      }
    },
    {
      "item": {
        "reference": "ReferralRequest/000000070001",
        "display": "UBRN 000000070001"
      }
    }
  ]
}
```

Note: The e-RS worklist returns a List of **references** (UBRNs) — not the full referral data. Each referral must then be fetched individually. BaRS returns the full ServiceRequest resources inline.

---

## Key Differences

| Dimension | e-RS (Worklist) | BaRS (ServiceRequest) |
|---|---|---|
| **Primary query axis** | **Organisation** — "show me all referrals for this service" | **Patient** — "show me all referrals for this patient" |
| **FHIR version** | STU3 (CareConnect profiles) | R4 (UKCore profiles) |
| **Interaction style** | Custom operation (`$ers.fetchworklist`) | Standard RESTful search (`GET`) |
| **User context** | Required (CIS2 role + org) | Not required (app-restricted) |
| **Use case** | Provider worklist management — "what's in my inbox?" | Patient-centric lookup — "what referrals exist for this patient?" |
| **Pagination** | Via worklist parameters | Standard FHIR Bundle pagination |
| **Filtering** | By worklist type (review, triage, A&G) | By patient identifier or ServiceRequest ID |

---

## Why Worklists Are an e-RS Concept

The "worklist" is a **user interface concept** specific to e-RS. It represents a clinician's inbox — the set of referrals awaiting action at their service. It is:

- **Role-scoped** — what you see depends on your role (clinician vs admin vs RAS reviewer)
- **Organisation-scoped** — shows referrals for your organisation/service only
- **Status-filtered** — grouped by action needed (for review, for triage, for A&G response)
- **User-driven** — requires a logged-in user with a specific business function

This is fundamentally a **presentation layer concern**, not a data query concern. The worklist is how e-RS organises work for humans — it is not a general-purpose API for retrieving referral data.

---

## Why GET /ServiceRequest Is the Strategic Direction

The BaRS `GET /ServiceRequest` approach is:

1. **Standard FHIR** — uses a standard RESTful search interaction, not a custom operation
2. **R4** — aligned with the strategic FHIR version (STU3 is legacy)
3. **System-to-system capable** — works with app-restricted auth, no human login required
4. **Composable** — can be combined with other FHIR queries (Appointment, DocumentReference, etc.)
5. **Not tied to a UI paradigm** — the consumer decides how to present the data, not the API

However, BaRS `GET /ServiceRequest` currently only supports **patient-scoped queries**. To support the provider use case (equivalent of "show me all referrals for my service"), changes are needed.

---

## What Needs to Change on the BaRS API

### The Gap: Organisation-Scoped Queries

Today, BaRS `GET /ServiceRequest` supports:

```
GET /ServiceRequest?patient:identifier=https://fhir.nhs.uk/Id/nhs-number|{nhs-number}
GET /ServiceRequest?_id={id}
```

To replace the e-RS worklist, it also needs to support querying **by receiving organisation or service** — "give me all ServiceRequests where my organisation is the performer."

### Proposed New Search Parameters

| Parameter | Type | Purpose | FHIR path |
|---|---|---|---|
| `performer` | reference | Filter by the organisation performing/receiving the referral | `ServiceRequest.performer` |
| `performer:identifier` | token | Filter by performer ODS code | `ServiceRequest.performer.identifier` |
| `status` | token | Filter by request status (active, completed, cancelled, etc.) | `ServiceRequest.status` |
| `authored` | date | Filter by when the request was created | `ServiceRequest.authoredOn` |
| `priority` | token | Filter by urgency/priority | `ServiceRequest.priority` |

### Example: Organisation-Scoped Query (Target State)

```http
GET /ServiceRequest?performer:identifier=https://fhir.nhs.uk/Id/ods-organization-code|R69&status=active HTTP/1.1
Host: int.api.service.nhs.uk
Accept: application/fhir+json
Authorization: Bearer eyJhbGciOi...
X-Request-Id: {uuid}
X-Correlation-Id: {uuid}
NHSD-End-User-Organisation: {base64-encoded org}
NHSD-Target-Identifier: {base64-encoded target}
```

This returns all active ServiceRequests where organisation `R69` is the performer — the equivalent of the e-RS "Referrals for Review" worklist, but as a standard FHIR search.

### Additional Considerations

| Consideration | Detail |
|---|---|
| **Pagination** | Organisation-scoped queries will return large result sets. Standard FHIR Bundle pagination (`_count`, `next` link) must be supported. |
| **Authorisation** | The requester should only see ServiceRequests for organisations they are authorised to access. The `NHSD-End-User-Organisation-ODS` header (or token claims) must be validated against the `performer` value. |
| **Sort** | Support for `_sort=authored` (newest first) to replicate the worklist ordering. |
| **Include** | Consider supporting `_include=ServiceRequest:patient` to return the Patient reference inline, avoiding N+1 lookups for each referral. |
| **Status mapping** | e-RS worklist types (REFERRALS_FOR_REVIEW, etc.) are a combination of `status` + `intent` + business state. The BaRS API may need a custom search parameter or extension to replicate this filtering without requiring the consumer to understand internal e-RS state machines. |
| **Volume** | Large providers may have thousands of active referrals. Efficient DynamoDB query patterns (GSI on performer + status) will be needed. |

---

## Migration Path

| Phase | Capability | e-RS equivalent |
|---|---|---|
| **Current** | `GET /ServiceRequest?patient:identifier={nhs-number}` | Patient-level referral lookup (not a worklist) |
| **Phase 1** | Add `performer:identifier` and `status` search params | Basic "referrals for my org" query |
| **Phase 2** | Add `_sort`, `_count`, pagination | Worklist-like ordering and paging |
| **Phase 3** | Add `_include=ServiceRequest:patient` | Efficient bulk retrieval without N+1 |
| **Future** | Custom search param or extension for business state filtering | Full worklist-type equivalence |

---

## Summary

| Question | Answer |
|---|---|
| Should we use e-RS worklists? | **No** — worklists are an e-RS UI concept tied to STU3 and CIS2 user context |
| Should we use BaRS `GET /ServiceRequest`? | **Yes** — standard FHIR R4, RESTful, composable, system-to-system capable |
| What's missing from BaRS today? | Organisation-scoped queries (`performer`), status filtering, pagination, sorting |
| How much work is this? | **Low for BaRS, significant for e-RS.** The BaRS API/standard changes are small — adding new search parameters to an existing endpoint is additive and non-breaking. The real effort is on the e-RS side: building the adapter that receives these queries, translates them into e-RS internal calls, transforms STU3 ReferralRequest data into R4 ServiceRequest responses, and handles the volume/performance characteristics of organisation-scoped queries across millions of referrals. |
| Is this a breaking change? | **No** — adding new search parameters is additive; existing patient-scoped queries continue to work |

---

## Surfacing Legacy e-RS Referral Data as FHIR R4

### The Problem

The e-RS system holds millions of historical referrals in its STU3/proprietary data model. These referrals are stored in the `ersdb` database and are only accessible today through the e-RS API (STU3 `ReferralRequest`).

If the strategic direction is `GET /ServiceRequest` (FHIR R4), there is a question: **how do consumers access legacy e-RS referral data through the new R4 interface?**

### Options for Legacy Data Access

#### Option 1: Facade / Adapter Service

A translation layer sits between the BaRS API and the e-RS backend, mapping STU3 `ReferralRequest` resources to R4 `ServiceRequest` on the fly.

```
Consumer → BaRS API (GET /ServiceRequest) → Adapter → e-RS API ($ers.fetchworklist or GET /ReferralRequest)
                                                ↓
                                          STU3 → R4 mapping
                                                ↓
                                          Return as R4 ServiceRequest
```

**How it works:**

1. Consumer calls `GET /ServiceRequest?performer:identifier=...&status=active`
2. The BaRS backend (or a dedicated adapter) calls the e-RS API internally to fetch matching referrals
3. The adapter transforms each STU3 `ReferralRequest` into an R4 `ServiceRequest` using a defined mapping
4. Results are returned to the consumer as a standard FHIR R4 Bundle

**Mapping: STU3 ReferralRequest → R4 ServiceRequest**

| STU3 ReferralRequest field | R4 ServiceRequest field | Notes |
|---|---|---|
| `ReferralRequest.id` (UBRN) | `ServiceRequest.identifier` | UBRN becomes a business identifier, not the resource ID |
| `ReferralRequest.status` | `ServiceRequest.status` | Values map: `active` → `active`, `completed` → `completed`, `cancelled` → `revoked` |
| `ReferralRequest.intent` | `ServiceRequest.intent` | Typically `order` |
| `ReferralRequest.priority` | `ServiceRequest.priority` | Direct map: `routine`, `urgent` |
| `ReferralRequest.subject` | `ServiceRequest.subject` | Patient reference (NHS Number) |
| `ReferralRequest.requester.agent` | `ServiceRequest.requester` | Requesting organisation/practitioner |
| `ReferralRequest.recipient` | `ServiceRequest.performer` | Receiving organisation |
| `ReferralRequest.authoredOn` | `ServiceRequest.authoredOn` | Date the referral was created |
| `ReferralRequest.specialty` | `ServiceRequest.code` | Clinical specialty (may need code system mapping) |
| `ReferralRequest.serviceRequested` | `ServiceRequest.category` | Service type requested |
| `ReferralRequest.description` | `ServiceRequest.note` | Free-text referral reason |
| (e-RS extension) shortlist | `ServiceRequest.locationReference` | Services on the patient's shortlist |

**Pros:**
- Single interface for consumers — R4 only, regardless of where the data lives
- Legacy data is accessible without consumers needing to integrate with e-RS directly
- Mapping is well-defined (STU3 → R4 mappings exist in the FHIR community)

**Cons:**
- Runtime translation adds latency
- Not all STU3 fields map cleanly to R4 (some e-RS extensions have no R4 equivalent)
- The adapter must handle e-RS authentication (CIS2 or app-restricted) on behalf of the caller
- Maintenance burden — changes to either the e-RS API or the R4 model require adapter updates

#### Option 2: Data Migration (Batch ETL)

Legacy e-RS referrals are migrated into the BaRS data store as R4 ServiceRequest resources. The migration runs as a batch ETL process, transforming and loading historical data.

```
e-RS Database (ersdb) → ETL Pipeline → BaRS Data Store (DynamoDB)
                              ↓
                    STU3 → R4 transformation
                    Deduplication
                    Validation
```

**How it works:**

1. A batch process reads referrals from `ersdb` (or via the e-RS API)
2. Each `ReferralRequest` is transformed to an R4 `ServiceRequest` using the mapping above
3. The R4 resources are written to the BaRS data store
4. The BaRS API serves them natively via `GET /ServiceRequest` — no runtime translation

**Pros:**
- No runtime overhead — data is pre-transformed and served natively
- Single data store, single query path
- Full FHIR R4 compliance — no facade leaking STU3 semantics

**Cons:**
- Large migration effort (millions of referrals)
- Point-in-time snapshot — ongoing sync needed for referrals still being updated in e-RS
- Data ownership question — who is the system of record? e-RS or BaRS?
- Storage cost for duplicating data

#### Option 3: Hybrid — New Data in BaRS, Legacy via Facade

New referrals (created after a cutover date) are created in BaRS natively as R4 ServiceRequests. Legacy referrals (before the cutover) are served via a facade to e-RS.

```
Consumer → BaRS API (GET /ServiceRequest)
                ↓
        ┌───────┴───────┐
        │               │
   BaRS Data Store   e-RS Adapter
   (new referrals)   (legacy referrals)
        │               │
        └───────┬───────┘
                ↓
         Merged R4 Bundle
```

**How it works:**

1. Consumer calls `GET /ServiceRequest?performer:identifier=...`
2. BaRS queries its own data store for post-cutover referrals
3. BaRS queries e-RS (via adapter) for pre-cutover referrals
4. Both result sets are merged, deduplicated, and returned as a single R4 Bundle
5. Over time, as legacy referrals are completed/cancelled, the e-RS adapter handles fewer and fewer requests
6. Eventually, all active data is in BaRS and the adapter can be retired

**Pros:**
- No big-bang migration needed
- New data is clean R4 from day one
- Legacy access degrades gracefully as referrals age out
- Clear system of record: BaRS for new, e-RS for legacy

**Cons:**
- Merge logic adds complexity (deduplication, consistent ordering)
- Two data sources in flight during transition period
- Consumer sees a single interface but backend complexity is significant
- Performance depends on the slower of the two backends

### Recommendation

**Option 3 (Hybrid)** is the most pragmatic approach for a phased transition:

1. Define a **cutover date** after which all new referrals are created via BaRS as R4 ServiceRequests
2. Build a **lightweight e-RS adapter** that translates legacy STU3 data to R4 on demand
3. Serve both through the same `GET /ServiceRequest` endpoint with merge logic
4. As legacy referrals reach terminal states (completed, cancelled), they fall out of active queries
5. Retire the adapter when the volume of active legacy referrals reaches zero

### Key Design Decisions Needed

| # | Decision | Impact |
|---|---|---|
| 1 | What is the cutover date? (When do new referrals stop going to e-RS?) | Determines how long the adapter runs |
| 2 | Do we migrate completed/cancelled referrals, or only active ones? | Affects migration volume and storage |
| 3 | How does the adapter authenticate to e-RS? (Service account? Delegated CIS2?) | Security model |
| 4 | Who is the system of record during the transition? (For updates — if a legacy referral is modified, where does the write go?) | Data integrity |
| 5 | How do we handle e-RS extensions that have no R4 equivalent? (Drop them? Map to R4 extensions?) | Data fidelity |
| 6 | Should legacy referrals carry a provenance indicator (e.g., `meta.source = "e-RS"`) so consumers know the origin? | Transparency |

---

## Open Questions

| # | Question | For |
|---|---|---|
| 1 | Should `performer` be the ODS code of the organisation, or the DoS service ID of the specific service? (e-RS uses service-level worklists, not org-level) | Architecture |
| 2 | Is `status` alone sufficient to replicate e-RS worklist types, or do we need a compound filter (status + category/intent)? | e-RS / BaRS team |
| 3 | Should the org-scoped query require CIS2 (user context) like e-RS, or is app-restricted sufficient? | Security |
| 4 | What volume of ServiceRequests per organisation should we design for? (Affects indexing strategy) | e-RS data team |
| 5 | Should the response include the full ServiceRequest or just summary fields (to reduce payload for large worklists)? | Architecture |
