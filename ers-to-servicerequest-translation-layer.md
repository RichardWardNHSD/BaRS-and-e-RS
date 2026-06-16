# e-RS to BaRS /ServiceRequest Translation Layer

## Purpose

This document details the design of a translation layer that sits between the e-RS database (`ersdb`) and a BaRS-aligned set of `/ServiceRequest` API operations. The translation layer enables consumers to interact with e-RS referral data through a standard FHIR R4 RESTful interface, without needing to understand the e-RS internal data model or STU3 API.

This is complementary to the strategic analysis in [worklists-vs-service-requests.md](./worklists-vs-service-requests.md) — that document covers *why*; this document covers *how*.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Consumer                                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  FHIR R4 (GET, POST, PUT, PATCH)
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       BaRS Proxy (Transport)                         │
│              Routes requests, enforces standard headers               │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Translation Layer (Facade)                        │
│                                                                       │
│  ┌───────────────┐  ┌───────────────────┐  ┌─────────────────────┐ │
│  │ Query Parser  │  │ Data Mapper       │  │ Response Assembler  │ │
│  │ (FHIR params  │  │ (ersdb → R4       │  │ (Bundle builder,    │ │
│  │  → DB query)  │  │  ServiceRequest)  │  │  pagination, links) │ │
│  └───────────────┘  └───────────────────┘  └─────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  SQL / internal query
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         ersdb (e-RS Database)                         │
│                                                                       │
│  Tables: referral, referral_status, patient, service, shortlist,     │
│          clinical_info, attachment, advice_request, ...               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Supported API Operations

The translation layer exposes the following FHIR R4 operations, each mapping to one or more internal e-RS database queries:

| Operation | HTTP | Path | Purpose |
|-----------|------|------|---------|
| **Search** | `GET` | `/ServiceRequest` | Retrieve referrals by patient, performer, status, date |
| **Read** | `GET` | `/ServiceRequest/{id}` | Retrieve a single referral by logical ID |
| **Create** | `POST` | `/ServiceRequest` | Create a new referral (write-through to ersdb) |
| **Update** | `PUT` | `/ServiceRequest/{id}` | Full update of an existing referral |
| **Patch** | `PATCH` | `/ServiceRequest/{id}` | Partial update (e.g. status change, triage outcome) |

### Search Parameters

| Parameter | Type | FHIR Path | ersdb Column(s) | Example |
|-----------|------|-----------|-----------------|---------|
| `_id` | token | `ServiceRequest.id` | `referral.ubrn` | `_id=000000070000` |
| `patient:identifier` | token | `ServiceRequest.subject.identifier` | `patient.nhs_number` | `patient:identifier=https://fhir.nhs.uk/Id/nhs-number\|9876543210` |
| `performer:identifier` | token | `ServiceRequest.performer.identifier` | `service.ods_code` | `performer:identifier=https://fhir.nhs.uk/Id/ods-organization-code\|R69` |
| `requester:identifier` | token | `ServiceRequest.requester.identifier` | `referral.referrer_ods_code` | `requester:identifier=https://fhir.nhs.uk/Id/ods-organization-code\|A1001` |
| `status` | token | `ServiceRequest.status` | `referral_status.current_status` | `status=active` |
| `intent` | token | `ServiceRequest.intent` | (derived) | `intent=order` |
| `priority` | token | `ServiceRequest.priority` | `referral.priority` | `priority=urgent` |
| `authored` | date | `ServiceRequest.authoredOn` | `referral.created_date` | `authored=ge2025-01-01` |
| `_sort` | special | — | ORDER BY clause | `_sort=-authored` |
| `_count` | number | — | LIMIT clause | `_count=50` |
| `_include` | special | — | JOIN Patient | `_include=ServiceRequest:patient` |

---

## Data Model Mapping

### Core Entity: `referral` → `ServiceRequest`

The primary mapping is from the `referral` table (and its related tables) to a FHIR R4 `ServiceRequest` resource conforming to the `UKCore-ServiceRequest` profile.

```
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│         ersdb tables            │         │    FHIR R4 ServiceRequest       │
├─────────────────────────────────┤         ├─────────────────────────────────┤
│ referral.ubrn                   │───────▶│ identifier[0].value             │
│ referral.version                │───────▶│ meta.versionId                  │
│ referral.last_updated           │───────▶│ meta.lastUpdated                │
│ referral_status.current_status  │───────▶│ status                          │
│ referral.intent_code            │───────▶│ intent                          │
│ referral.priority               │───────▶│ priority                        │
│ referral.specialty_code         │───────▶│ code.coding[0]                  │
│ referral.service_type_code      │───────▶│ category[0].coding[0]           │
│ patient.nhs_number              │───────▶│ subject.identifier.value        │
│ referral.referrer_ods_code      │───────▶│ requester.identifier.value      │
│ service.ods_code                │───────▶│ performer[0].identifier.value   │
│ referral.created_date           │───────▶│ authoredOn                      │
│ referral.clinical_info_summary  │───────▶│ note[0].text                    │
│ referral.reason_code            │───────▶│ reasonCode[0].coding[0]         │
│ shortlist entries               │───────▶│ locationReference[]              │
│ referral.advice_request_flag    │───────▶│ category (A&G indicator)        │
└─────────────────────────────────┘         └─────────────────────────────────┘
```

### Detailed Field Mapping

| # | ersdb Source | ersdb Type | R4 Target | R4 Type | Transformation |
|---|---|---|---|---|---|
| 1 | `referral.ubrn` | VARCHAR(12) | `ServiceRequest.identifier[0]` | Identifier | Wrap as `{system: "https://fhir.nhs.uk/Id/UBRN", value: "{ubrn}"}` |
| 2 | (generated UUID) | — | `ServiceRequest.id` | id | Generate a stable UUID from UBRN (deterministic hash or lookup table) |
| 3 | `referral.version` | INT | `ServiceRequest.meta.versionId` | string | Cast to string |
| 4 | `referral.last_updated` | TIMESTAMP | `ServiceRequest.meta.lastUpdated` | instant | Format as ISO 8601 with timezone |
| 5 | `referral_status.current_status` | VARCHAR | `ServiceRequest.status` | code | Map via status translation table (see below) |
| 6 | `referral.intent_code` | VARCHAR | `ServiceRequest.intent` | code | Default `order`; A&G requests map to `proposal` |
| 7 | `referral.priority` | VARCHAR | `ServiceRequest.priority` | code | Direct map: `ROUTINE` → `routine`, `URGENT` → `urgent`, `TWO_WEEK_WAIT` → `urgent` |
| 8 | `referral.specialty_code` | VARCHAR | `ServiceRequest.code.coding[0]` | CodeableConcept | Map to SNOMED or NHS specialty code system |
| 9 | `referral.service_type_code` | VARCHAR | `ServiceRequest.category[0]` | CodeableConcept | Map to BaRS use-case category code system |
| 10 | `patient.nhs_number` | VARCHAR(10) | `ServiceRequest.subject.identifier` | Identifier | `{system: "https://fhir.nhs.uk/Id/nhs-number", value: "{nhs_number}"}` |
| 11 | `patient.family_name`, `patient.given_name` | VARCHAR | `ServiceRequest.subject.display` | string | Concatenate: `"{given_name} {family_name}"` |
| 12 | `referral.referrer_ods_code` | VARCHAR | `ServiceRequest.requester.identifier` | Identifier | `{system: "https://fhir.nhs.uk/Id/ods-organization-code", value: "{ods}"}` |
| 13 | `organisation.name` (referrer) | VARCHAR | `ServiceRequest.requester.display` | string | Organisation display name |
| 14 | `service.ods_code` | VARCHAR | `ServiceRequest.performer[0].identifier` | Identifier | `{system: "https://fhir.nhs.uk/Id/ods-organization-code", value: "{ods}"}` |
| 15 | `service.name` | VARCHAR | `ServiceRequest.performer[0].display` | string | Service display name |
| 16 | `referral.created_date` | TIMESTAMP | `ServiceRequest.authoredOn` | dateTime | Format as ISO 8601 |
| 17 | `referral.clinical_info_summary` | TEXT | `ServiceRequest.note[0].text` | string | Direct copy (may need HTML → plain text) |
| 18 | `referral.reason_code` | VARCHAR | `ServiceRequest.reasonCode[0].coding[0]` | CodeableConcept | Map to SNOMED CT |
| 19 | `shortlist.service_id` (multiple) | FK | `ServiceRequest.locationReference[]` | Reference | One reference per shortlisted service |
| 20 | `referral.appointment_id` | FK | `ServiceRequest.supportingInfo[]` | Reference | Reference to related Appointment if booked |

---

## Status Translation

The e-RS internal status model is richer than the FHIR `ServiceRequest.status` value set. The translation layer maps e-RS statuses to valid R4 codes and preserves the original e-RS status as an extension for consumers that need it.

### Status Mapping Table

| e-RS Internal Status | R4 `ServiceRequest.status` | R4 `ServiceRequest.intent` | Notes |
|---|---|---|---|
| `REFERRAL_CREATED` | `draft` | `order` | Referral exists but not yet sent |
| `AWAITING_TRIAGE` | `active` | `order` | Sent to provider, awaiting clinical review |
| `TRIAGED_PROVIDER_RESPONSE` | `active` | `order` | Triage outcome recorded |
| `AWAITING_BOOKING` | `active` | `order` | Accepted, awaiting appointment slot |
| `BOOKED` | `active` | `order` | Appointment confirmed |
| `ACCEPTED_INTERIM` | `active` | `order` | Interim acceptance pending full review |
| `ASSESSMENT_STARTED` | `active` | `order` | Clinical assessment in progress |
| `DID_NOT_ATTEND` | `active` | `order` | Patient DNA — still active referral |
| `DEFERRED_TO_PROVIDER` | `active` | `order` | Deferred for provider to action |
| `CANCELLED_BY_REFERRER` | `revoked` | `order` | Referrer withdrew the referral |
| `CANCELLED_BY_PROVIDER` | `revoked` | `order` | Provider rejected/cancelled |
| `REJECTED` | `revoked` | `order` | Provider rejected at triage |
| `COMPLETED` | `completed` | `order` | Referral episode complete |
| `DISCHARGED` | `completed` | `order` | Patient discharged from pathway |
| `ADVICE_REQUEST_CREATED` | `active` | `proposal` | A&G request raised |
| `ADVICE_RESPONSE_SENT` | `completed` | `proposal` | A&G response provided |
| `ADVICE_CANCELLED` | `revoked` | `proposal` | A&G request withdrawn |
| `CONVERTED_TO_REFERRAL` | `completed` | `proposal` | A&G converted to a full referral |

### Preserving e-RS Detail

The original e-RS status is preserved in an extension on the ServiceRequest:

```json
{
  "url": "https://fhir.nhs.uk/StructureDefinition/Extension-eRS-ReferralStatus",
  "valueCoding": {
    "system": "https://fhir.nhs.uk/CodeSystem/eRS-ReferralStatus",
    "code": "AWAITING_TRIAGE",
    "display": "Awaiting Triage"
  }
}
```

This allows consumers to query using standard FHIR status codes (`status=active`) while still having access to the granular e-RS business state if needed.

---

## Query Translation

The Query Parser component translates incoming FHIR search parameters into efficient database queries against `ersdb`.

### Example: Organisation-Scoped Search

**Incoming FHIR request:**

```http
GET /ServiceRequest?performer:identifier=https://fhir.nhs.uk/Id/ods-organization-code|R69&status=active&_sort=-authored&_count=25 HTTP/1.1
```

**Translated database query (conceptual):**

```sql
SELECT
    r.ubrn,
    r.version,
    r.last_updated,
    r.intent_code,
    r.priority,
    r.specialty_code,
    r.service_type_code,
    r.created_date,
    r.clinical_info_summary,
    r.reason_code,
    r.referrer_ods_code,
    rs.current_status,
    p.nhs_number,
    p.family_name,
    p.given_name,
    s.ods_code AS performer_ods,
    s.name AS performer_name,
    o.name AS requester_name
FROM referral r
JOIN referral_status rs ON r.ubrn = rs.ubrn AND rs.is_current = true
JOIN patient p ON r.patient_id = p.id
JOIN service s ON r.receiving_service_id = s.id
JOIN organisation o ON r.referrer_ods_code = o.ods_code
WHERE s.ods_code = 'R69'
  AND rs.current_status IN (
      'AWAITING_TRIAGE', 'TRIAGED_PROVIDER_RESPONSE', 'AWAITING_BOOKING',
      'BOOKED', 'ACCEPTED_INTERIM', 'ASSESSMENT_STARTED', 'DID_NOT_ATTEND',
      'DEFERRED_TO_PROVIDER', 'ADVICE_REQUEST_CREATED'
  )
ORDER BY r.created_date DESC
LIMIT 25
OFFSET 0;
```

**Key translation logic:**

| FHIR Parameter | Translation Rule |
|---|---|
| `performer:identifier=...ods-organization-code\|R69` | `WHERE s.ods_code = 'R69'` |
| `status=active` | Expand to all e-RS statuses that map to `active` (see status table) |
| `_sort=-authored` | `ORDER BY r.created_date DESC` |
| `_count=25` | `LIMIT 25` |

### Example: Patient-Scoped Search

**Incoming FHIR request:**

```http
GET /ServiceRequest?patient:identifier=https://fhir.nhs.uk/Id/nhs-number|9876543210 HTTP/1.1
```

**Translated database query:**

```sql
SELECT r.*, rs.current_status, p.*, s.*, o.*
FROM referral r
JOIN referral_status rs ON r.ubrn = rs.ubrn AND rs.is_current = true
JOIN patient p ON r.patient_id = p.id
JOIN service s ON r.receiving_service_id = s.id
JOIN organisation o ON r.referrer_ods_code = o.ods_code
WHERE p.nhs_number = '9876543210'
ORDER BY r.created_date DESC;
```

### Example: Single Referral Read

**Incoming FHIR request:**

```http
GET /ServiceRequest/000000070000 HTTP/1.1
```

**Translated database query:**

```sql
SELECT r.*, rs.current_status, p.*, s.*, o.*
FROM referral r
JOIN referral_status rs ON r.ubrn = rs.ubrn AND rs.is_current = true
JOIN patient p ON r.patient_id = p.id
JOIN service s ON r.receiving_service_id = s.id
JOIN organisation o ON r.referrer_ods_code = o.ods_code
WHERE r.ubrn = '000000070000';
```

---

## Write Operations (Create, Update, Patch)

Write operations reverse the mapping — accepting FHIR R4 `ServiceRequest` payloads and translating them into `ersdb` writes.

### POST /ServiceRequest (Create)

Creates a new referral in `ersdb`. The translation layer:

1. Validates the incoming ServiceRequest against the UKCore-ServiceRequest profile
2. Extracts the patient NHS Number from `subject.identifier` → looks up or creates `patient` record
3. Resolves the performer ODS code → looks up `service` record
4. Maps R4 fields back to `referral` table columns
5. Generates a new UBRN
6. Inserts into `referral` and `referral_status` tables
7. Returns the created resource with `id`, `meta.versionId`, and `Location` header

**Reverse mapping (R4 → ersdb):**

| R4 Field | ersdb Target | Transformation |
|---|---|---|
| `ServiceRequest.subject.identifier.value` | `referral.patient_id` (FK via lookup) | Look up patient by NHS Number |
| `ServiceRequest.performer[0].identifier.value` | `referral.receiving_service_id` (FK via lookup) | Look up service by ODS code |
| `ServiceRequest.requester.identifier.value` | `referral.referrer_ods_code` | Direct write |
| `ServiceRequest.priority` | `referral.priority` | `routine` → `ROUTINE`, `urgent` → `URGENT` |
| `ServiceRequest.code.coding[0].code` | `referral.specialty_code` | Map from SNOMED/NHS code |
| `ServiceRequest.category[0].coding[0].code` | `referral.service_type_code` | Map from BaRS category |
| `ServiceRequest.reasonCode[0].coding[0]` | `referral.reason_code` | SNOMED CT code |
| `ServiceRequest.note[0].text` | `referral.clinical_info_summary` | Direct write |
| `ServiceRequest.intent` | `referral.intent_code` | `order` → standard referral; `proposal` → A&G |

### PUT /ServiceRequest/{id} (Update)

Full replacement of an existing referral. The translation layer:

1. Validates the incoming resource
2. Checks `meta.versionId` matches current version (optimistic locking)
3. Maps all fields back to `ersdb` columns
4. Updates the `referral` record
5. If status has changed, inserts a new `referral_status` record and marks previous as non-current
6. Increments version
7. Returns the updated resource

### PATCH /ServiceRequest/{id} (Partial Update)

Supports JSON Patch (`application/json-patch+json`) for targeted changes. Common patch operations:

**Status change (e.g. triage acceptance):**

```json
[
  {
    "op": "replace",
    "path": "/status",
    "value": "active"
  },
  {
    "op": "add",
    "path": "/extension/-",
    "value": {
      "url": "https://fhir.nhs.uk/StructureDefinition/Extension-eRS-ReferralStatus",
      "valueCoding": {
        "system": "https://fhir.nhs.uk/CodeSystem/eRS-ReferralStatus",
        "code": "TRIAGED_PROVIDER_RESPONSE"
      }
    }
  }
]
```

**Priority escalation:**

```json
[
  {
    "op": "replace",
    "path": "/priority",
    "value": "urgent"
  }
]
```

---

## Response Assembly

The Response Assembler constructs valid FHIR R4 Bundle responses from the mapped data.

### Search Response Structure

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 142,
  "link": [
    {
      "relation": "self",
      "url": "https://api.service.nhs.uk/booking-and-referral/FHIR/R4/ServiceRequest?performer:identifier=https://fhir.nhs.uk/Id/ods-organization-code|R69&status=active&_count=25"
    },
    {
      "relation": "next",
      "url": "https://api.service.nhs.uk/booking-and-referral/FHIR/R4/ServiceRequest?performer:identifier=https://fhir.nhs.uk/Id/ods-organization-code|R69&status=active&_count=25&_offset=25"
    }
  ],
  "entry": [
    {
      "fullUrl": "https://api.service.nhs.uk/booking-and-referral/FHIR/R4/ServiceRequest/000000070000",
      "resource": {
        "resourceType": "ServiceRequest",
        "id": "000000070000",
        "meta": {
          "versionId": "7",
          "lastUpdated": "2025-06-10T14:30:00+01:00",
          "profile": [
            "https://fhir.hl7.org.uk/StructureDefinition/UKCore-ServiceRequest"
          ]
        },
        "extension": [
          {
            "url": "https://fhir.nhs.uk/StructureDefinition/Extension-eRS-ReferralStatus",
            "valueCoding": {
              "system": "https://fhir.nhs.uk/CodeSystem/eRS-ReferralStatus",
              "code": "AWAITING_TRIAGE",
              "display": "Awaiting Triage"
            }
          }
        ],
        "identifier": [
          {
            "system": "https://fhir.nhs.uk/Id/UBRN",
            "value": "000000070000"
          }
        ],
        "status": "active",
        "intent": "order",
        "priority": "routine",
        "code": {
          "coding": [
            {
              "system": "https://fhir.nhs.uk/CodeSystem/eRS-Specialty",
              "code": "CARDIOLOGY",
              "display": "Cardiology"
            }
          ]
        },
        "subject": {
          "identifier": {
            "system": "https://fhir.nhs.uk/Id/nhs-number",
            "value": "9876543210"
          },
          "display": "John Smith"
        },
        "authoredOn": "2025-06-09T09:15:00+01:00",
        "requester": {
          "identifier": {
            "system": "https://fhir.nhs.uk/Id/ods-organization-code",
            "value": "A1001"
          },
          "display": "Anytown GP Practice"
        },
        "performer": [
          {
            "identifier": {
              "system": "https://fhir.nhs.uk/Id/ods-organization-code",
              "value": "R69"
            },
            "display": "Anytown General Hospital - Cardiology"
          }
        ],
        "note": [
          {
            "text": "Patient presenting with intermittent chest pain on exertion. ECG normal. Requesting cardiology assessment."
          }
        ]
      },
      "search": {
        "mode": "match"
      }
    }
  ]
}
```

### Pagination

The translation layer uses offset-based pagination:

| Parameter | Default | Max | Behaviour |
|-----------|---------|-----|-----------|
| `_count` | 25 | 100 | Number of results per page |
| `_offset` | 0 | — | Starting position in result set |

The `Bundle.total` reflects the total matching count (requires a `COUNT(*)` query). The `Bundle.link` array contains `self`, `next`, and optionally `previous` links.

---

## Error Handling

The translation layer returns standard FHIR OperationOutcome resources for all error conditions:

| HTTP Status | Condition | OperationOutcome.issue.code |
|---|---|---|
| 400 | Invalid search parameter or value | `invalid` |
| 401 | Missing or invalid auth token | `security` |
| 403 | Requester not authorised for the target performer org | `forbidden` |
| 404 | ServiceRequest ID not found in ersdb | `not-found` |
| 409 | Version conflict on update (stale `versionId`) | `conflict` |
| 422 | Resource fails profile validation | `unprocessable` |
| 500 | ersdb connection failure or unexpected error | `exception` |
| 503 | ersdb unavailable or translation layer at capacity | `transient` |

### Example Error Response

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "not-found",
      "details": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/CodeSystem/http-error-codes",
            "code": "RESOURCE_NOT_FOUND"
          }
        ]
      },
      "diagnostics": "No referral found with UBRN 000000099999"
    }
  ]
}
```

---

## Authorisation Model

The translation layer enforces access control at the query level:

| Check | Mechanism | Enforcement |
|---|---|---|
| **Caller identity** | Signed JWT (app-restricted) or CIS2 token (user-restricted) | BaRS Proxy validates token before forwarding |
| **Organisation scope** | `NHSD-End-User-Organisation` header | Translation layer validates that the requesting org is authorised to query the target performer |
| **Data filtering** | Row-level security | Query results are filtered to only return referrals where the requesting org is either the referrer or the performer |
| **Sensitive fields** | Field-level redaction | Some clinical details may be withheld based on caller role (e.g. admin vs clinician) |

### Authorisation Flow

```
1. Consumer presents signed JWT + NHSD-End-User-Organisation header
2. BaRS Proxy validates JWT signature and expiry
3. Translation layer extracts org identity from header/token claims
4. Query is scoped: WHERE (s.ods_code = '{caller_ods}' OR r.referrer_ods_code = '{caller_ods}')
5. Results returned only contain referrals visible to the calling organisation
```

---

## Performance Considerations

### Database Indexing Requirements

Organisation-scoped queries will be the primary access pattern. The following indexes are critical:

| Index | Columns | Purpose |
|---|---|---|
| `idx_referral_service_status` | `receiving_service_id, referral_status.current_status, created_date DESC` | Org-scoped active referral queries |
| `idx_referral_patient` | `patient_id, created_date DESC` | Patient-scoped queries |
| `idx_referral_ubrn` | `ubrn` (PK) | Single referral read |
| `idx_referral_referrer` | `referrer_ods_code, created_date DESC` | Requester-scoped queries |
| `idx_referral_status_current` | `ubrn, is_current` | Current status lookup |

### Caching Strategy

| Layer | Cache Type | TTL | Invalidation |
|---|---|---|---|
| **Organisation/Service metadata** | In-memory (LRU) | 1 hour | On ODS lookup miss |
| **Code system mappings** | In-memory (static) | Until deploy | Immutable per release |
| **Query results** | None | — | Referral data changes too frequently for result caching |
| **Patient demographics** | Short-lived (LRU) | 5 minutes | Reduces patient table joins for repeated queries |

### Volume Estimates

| Metric | Estimate | Implication |
|---|---|---|
| Total referrals in ersdb | ~50 million+ | Indexing and pagination are essential |
| Active referrals at any time | ~2–3 million | Org-scoped active queries must be fast |
| Referrals per large provider org | 10,000–50,000 active | Pagination mandatory; `_count` default of 25 |
| Read:Write ratio | ~95:5 | Optimise for read-heavy workload |
| Peak query rate | ~500 req/s (estimated) | Connection pooling and read replicas needed |

### Read Replica Strategy

For read-heavy org-scoped queries, the translation layer should route `GET` operations to a read replica of `ersdb`:

```
GET /ServiceRequest  → Read replica (eventual consistency, ~1s lag acceptable)
POST /ServiceRequest → Primary (strong consistency)
PUT /ServiceRequest  → Primary (strong consistency)
PATCH /ServiceRequest → Primary (strong consistency)
```

---

## Handling e-RS Concepts with No Direct R4 Equivalent

Several e-RS concepts do not map cleanly to the R4 ServiceRequest resource. The translation layer handles these via extensions, contained resources, or related resources.

| e-RS Concept | R4 Approach | Detail |
|---|---|---|
| **UBRN** (Unique Booking Reference Number) | `ServiceRequest.identifier` | Business identifier with system `https://fhir.nhs.uk/Id/UBRN` |
| **Shortlist** (patient's choice of services) | `ServiceRequest.locationReference[]` | Each shortlisted service as a HealthcareService reference |
| **Worklist type** (REFERRALS_FOR_REVIEW, etc.) | Extension + status/category combination | Custom extension preserves the concept; consumers derive worklist type from `status` + `intent` + e-RS status extension |
| **Clinical attachments** | `ServiceRequest.supportingInfo[]` → `DocumentReference` | Separate resources referenced from ServiceRequest |
| **Triage response / outcome** | Extension or `Task` resource | Triage outcome stored as an extension on ServiceRequest or as a linked Task |
| **Named clinician (recipient)** | `ServiceRequest.performer[]` with Practitioner reference | If referral is to a named clinician, performer includes both org and practitioner |
| **Advice & Guidance conversation** | `ServiceRequest` (intent=proposal) + linked `Communication` resources | The A&G request is the ServiceRequest; messages are Communication resources |
| **Appointment slot** | `ServiceRequest.supportingInfo[]` → `Appointment` reference | Links to the booked appointment resource |
| **Referral letter / clinical info** | `ServiceRequest.supportingInfo[]` → `DocumentReference` | Binary attachment with metadata |
| **CIS2 user context** (referring clinician) | `ServiceRequest.requester` as PractitionerRole | When user-level identity is available, map to PractitionerRole with org + role |

### Extension Definitions

The translation layer uses the following custom extensions (to be formally published):

```json
[
  {
    "url": "https://fhir.nhs.uk/StructureDefinition/Extension-eRS-ReferralStatus",
    "description": "The granular e-RS referral status, preserving detail beyond the R4 status value set",
    "context": "ServiceRequest",
    "type": "Coding"
  },
  {
    "url": "https://fhir.nhs.uk/StructureDefinition/Extension-eRS-WorklistType",
    "description": "The e-RS worklist category this referral belongs to",
    "context": "ServiceRequest",
    "type": "CodeableConcept"
  },
  {
    "url": "https://fhir.nhs.uk/StructureDefinition/Extension-eRS-TriageOutcome",
    "description": "Triage decision made by the receiving clinician",
    "context": "ServiceRequest",
    "type": "CodeableConcept"
  },
  {
    "url": "https://fhir.nhs.uk/StructureDefinition/Extension-eRS-ReferralSource",
    "description": "Indicates this ServiceRequest was translated from legacy e-RS data",
    "context": "ServiceRequest.meta",
    "type": "uri"
  }
]
```

---

## Implementation Components

### Component Breakdown

```
translation-layer/
├── api/
│   ├── routes/
│   │   ├── search-service-request.ts      # GET /ServiceRequest (search)
│   │   ├── read-service-request.ts        # GET /ServiceRequest/{id}
│   │   ├── create-service-request.ts      # POST /ServiceRequest
│   │   ├── update-service-request.ts      # PUT /ServiceRequest/{id}
│   │   └── patch-service-request.ts       # PATCH /ServiceRequest/{id}
│   ├── middleware/
│   │   ├── auth-validator.ts              # JWT / CIS2 token validation
│   │   ├── org-scope-enforcer.ts          # Organisation-level access control
│   │   └── request-logger.ts              # Audit logging
│   └── validators/
│       ├── search-param-validator.ts      # FHIR search parameter parsing
│       └── resource-validator.ts          # UKCore profile validation
├── mapping/
│   ├── ersdb-to-r4.ts                    # Forward mapping (read path)
│   ├── r4-to-ersdb.ts                    # Reverse mapping (write path)
│   ├── status-mapper.ts                  # Status translation table
│   ├── code-system-mapper.ts             # Specialty, category, reason code maps
│   └── extension-builder.ts             # Custom extension construction
├── query/
│   ├── query-builder.ts                  # FHIR params → SQL query construction
│   ├── pagination.ts                     # Offset/limit + Bundle link generation
│   └── connection-pool.ts                # ersdb connection management
├── response/
│   ├── bundle-assembler.ts               # Constructs FHIR Bundle searchset
│   ├── operation-outcome-factory.ts      # Error response construction
│   └── etag-generator.ts                 # ETag from versionId for caching headers
└── config/
    ├── code-systems.json                 # Static code system mapping tables
    ├── status-map.json                   # e-RS status → R4 status mapping
    └── feature-flags.json                # Toggles for phased rollout
```

### Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Resource ID strategy** | Use UBRN as ServiceRequest.id | UBRNs are unique, immutable, and already the primary identifier in e-RS. Avoids maintaining a separate UUID ↔ UBRN lookup table. |
| **Version tracking** | Use `referral.version` as `meta.versionId` | Enables optimistic locking and change detection without additional infrastructure |
| **Profile** | `UKCore-ServiceRequest` | Strategic UK FHIR profile; aligns with BaRS standard |
| **Pagination** | Offset-based (not cursor-based) | Simpler to implement against SQL; acceptable for expected page sizes (≤100) |
| **Write-through vs event-sourced** | Write-through to ersdb | Translation layer writes directly to ersdb to maintain it as system of record during transition |
| **A&G handling** | Same resource type, distinguished by `intent=proposal` | Keeps the API surface simple; A&G is a ServiceRequest with different intent |

---

## Deployment Model

The translation layer is deployed as a discrete service within the e-RS infrastructure boundary (since it requires direct database access):

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           NHS Spine / APIM                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────┐         ┌─────────────────────────────────────────────┐   │
│  │  BaRS Proxy  │────────▶│     e-RS Infrastructure Boundary            │   │
│  │  (APIM)      │         │                                             │   │
│  └──────────────┘         │  ┌───────────────────────────────────┐      │   │
│                            │  │    Translation Layer Service       │      │   │
│                            │  │    (Containerised / Lambda)        │      │   │
│                            │  │                                   │      │   │
│                            │  │  ┌──────┐ ┌──────┐ ┌──────┐     │      │   │
│                            │  │  │Query │ │Mapper│ │Resp. │     │      │   │
│                            │  │  │Parser│ │      │ │Assem.│     │      │   │
│                            │  │  └──┬───┘ └──┬───┘ └──┬───┘     │      │   │
│                            │  └─────┼────────┼────────┼──────────┘      │   │
│                            │        │        │        │                   │   │
│                            │        ▼        ▼        ▼                   │   │
│                            │  ┌───────────────────────────────────┐      │   │
│                            │  │         ersdb (PostgreSQL)         │      │   │
│                            │  │    Primary + Read Replica          │      │   │
│                            │  └───────────────────────────────────┘      │   │
│                            └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why Inside the e-RS Boundary

- Direct database access avoids the overhead of calling the e-RS STU3 API and re-translating
- Avoids the authentication complexity of e-RS API calls (CIS2 service accounts)
- Gives full control over query performance (indexes, read replicas)
- Keeps the translation as close to the data as possible

### Deployment Options

| Option | Pros | Cons |
|---|---|---|
| **Containerised service (ECS/EKS)** | Always-on, connection pooling, predictable latency | Infrastructure cost even at low traffic |
| **Lambda + RDS Proxy** | Scales to zero, low cost at low traffic | Cold starts, connection management complexity |
| **Sidecar to existing e-RS** | Minimal new infrastructure, shares existing DB connections | Coupling to e-RS deployment lifecycle |

**Recommendation:** Containerised service (ECS Fargate) with connection pooling via PgBouncer. This provides predictable latency for org-scoped queries that may scan thousands of rows, avoids Lambda cold-start issues, and allows connection reuse across requests.

---

## Phased Rollout

| Phase | Scope | Operations | Search Params | Notes |
|---|---|---|---|---|
| **1 — Read-only, patient-scoped** | Single referral read + patient search | `GET /ServiceRequest/{id}`, `GET /ServiceRequest?patient:identifier=...` | `_id`, `patient:identifier` | Proves the mapping; low risk; read replica only |
| **2 — Read-only, org-scoped** | Organisation-level queries | `GET /ServiceRequest?performer:identifier=...` | `performer:identifier`, `status`, `_sort`, `_count` | Replaces worklist polling pattern |
| **3 — Write operations** | Create, update, patch | `POST`, `PUT`, `PATCH` | — | Requires write access to primary ersdb; higher risk |
| **4 — Full feature parity** | Includes, advanced search, A&G | All | `_include`, `requester:identifier`, `intent`, `authored`, `priority` | Feature-complete translation layer |

### Feature Flags

Each phase is gated behind feature flags, allowing gradual enablement:

```json
{
  "read_single_enabled": true,
  "read_patient_search_enabled": true,
  "read_org_search_enabled": false,
  "write_create_enabled": false,
  "write_update_enabled": false,
  "write_patch_enabled": false,
  "include_patient_enabled": false,
  "advanced_search_enabled": false
}
```

---

## Testing Strategy

| Test Type | Coverage | Approach |
|---|---|---|
| **Unit tests** | Individual mappers (status, code systems, field mapping) | Mock ersdb rows → assert R4 output matches expected |
| **Integration tests** | Full request → response cycle | Test database with known referral data; assert Bundle structure |
| **Contract tests** | FHIR profile compliance | Validate every response against UKCore-ServiceRequest profile using HAPI FHIR Validator |
| **Performance tests** | Org-scoped queries at scale | Load test with realistic data volumes (50K+ referrals per org) |
| **Regression tests** | Mapping correctness across e-RS status transitions | Known referral journeys replayed; assert correct R4 status at each stage |

---

## Open Questions

| # | Question | Impact | Owner |
|---|---|---|---|
| 1 | What is the exact ersdb schema? (Table names, column types, relationships) | Blocks detailed query implementation | e-RS team |
| 2 | Is direct DB access permitted, or must we go via an internal e-RS data access layer? | Determines deployment architecture | e-RS team / IG |
| 3 | Are there referral states in ersdb not covered by the published e-RS API documentation? | May reveal unmapped statuses | e-RS team |
| 4 | How is the UBRN generated? Can the translation layer generate them for new referrals? | Affects create flow | e-RS team |
| 5 | What clinical coding systems does e-RS use for specialty and reason codes? | Affects code system mapping tables | e-RS team |
| 6 | Is there an existing read replica, or does one need provisioning? | Infrastructure lead time | e-RS DBA |
| 7 | What is the e-RS data retention policy? Are old referrals archived/purged? | Affects query scope and index design | e-RS team |
| 8 | How should the translation layer handle in-flight referrals during the write-operation rollout? (Referrals being modified via both the legacy e-RS API and the new R4 interface simultaneously) | Data consistency during transition | Architecture |
| 9 | Should `meta.source` indicate the origin system for translated resources? | Provenance transparency | BaRS team |
| 10 | What SLA is expected for org-scoped queries? (P95 latency target) | Drives index and caching decisions | Product |

---

## Summary

This translation layer provides a standards-compliant FHIR R4 interface over the existing e-RS database, enabling consumers to interact with referral data via BaRS-aligned `/ServiceRequest` operations without coupling to the legacy STU3 API or worklist paradigm. It is designed to be:

- **Incrementally deployable** — read-only operations first, writes later
- **Standards-compliant** — UKCore-ServiceRequest profile, standard FHIR search semantics
- **Transparent** — e-RS-specific detail preserved via extensions, not lost in translation
- **Performant** — indexed queries, read replicas, connection pooling for high-volume org-scoped access
- **Transitional** — supports the hybrid model where this facade serves legacy data while new referrals are created in a purpose-built Referral Service

The long-term trajectory is for this translation layer to handle a diminishing volume of queries as active referrals naturally migrate to the new Referral Service. Eventually, when all active referrals exist natively in R4, the facade can be retired.
