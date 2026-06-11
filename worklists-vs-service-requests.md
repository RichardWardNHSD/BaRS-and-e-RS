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
| How much work is this? | Moderate — new search parameters on an existing endpoint, plus backend query optimisation for large result sets |
| Is this a breaking change? | **No** — adding new search parameters is additive; existing patient-scoped queries continue to work |

---

## Open Questions

| # | Question | For |
|---|---|---|
| 1 | Should `performer` be the ODS code of the organisation, or the DoS service ID of the specific service? (e-RS uses service-level worklists, not org-level) | Architecture |
| 2 | Is `status` alone sufficient to replicate e-RS worklist types, or do we need a compound filter (status + category/intent)? | e-RS / BaRS team |
| 3 | Should the org-scoped query require CIS2 (user context) like e-RS, or is app-restricted sufficient? | Security |
| 4 | What volume of ServiceRequests per organisation should we design for? (Affects indexing strategy) | e-RS data team |
| 5 | Should the response include the full ServiceRequest or just summary fields (to reduce payload for large worklists)? | Architecture |
