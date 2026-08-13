# Gap Analysis: FHIR STU3 ReferralRequest vs FHIR R4 ServiceRequest

## Introduction

### Why ReferralRequest Was Replaced by ServiceRequest

In FHIR STU3, the `ReferralRequest` resource was purpose-built for capturing referrals and transfers of care between practitioners or organisations. Alongside it, `ProcedureRequest` handled requests for diagnostic or therapeutic procedures. In practice, the boundary between these two resources was blurred — many clinical scenarios could reasonably be modelled using either, and implementers struggled to determine which was appropriate.

The HL7 FHIR community recognised that referrals, procedure orders, diagnostic requests, and other service-oriented requests share a common underlying pattern: one party asks another to perform a service. The distinction between "I'm asking you to use your judgement" (referral) and "I'm telling you what to do" (procedure order) was better represented as a property of the request (its `intent`) than as a fundamentally different resource type.

With FHIR R4, HL7 merged `ReferralRequest` and `ProcedureRequest` into a single, unified resource: `ServiceRequest`. This consolidation:

- Eliminates ambiguity about which resource to use for a given scenario
- Provides a single search target for systems that need to find "all requests for services"
- Simplifies workflow engines that process request resources
- Enables richer expression through a broader set of elements (e.g., `doNotPerform`, `bodySite`, `specimen`)
- Aligns with the FHIR Request Pattern, making the resource consistent with other R4 request resources

For the NHS e-Referral Service (e-RS), which built its FHIR API on STU3 `ReferralRequest`, this transition means that any move to FHIR R4 requires mapping from `ReferralRequest` to `ServiceRequest` — understanding what maps directly, what has been renamed, what has been restructured, and what requires e-RS-specific extensions to be re-evaluated.

---

## Element-by-Element Gap Analysis

### Elements That Map Directly (Minimal Change)

| STU3 ReferralRequest | R4 ServiceRequest | Change Type | Notes |
|---|---|---|---|
| `identifier` | `identifier` | None | Same semantics, same type |
| `basedOn` | `basedOn` | Reference targets changed | STU3: ReferralRequest, CarePlan, ProcedureRequest → R4: CarePlan, ServiceRequest, MedicationRequest |
| `replaces` | `replaces` | Reference target changed | STU3: ReferralRequest → R4: ServiceRequest |
| `status` | `status` | Value set refined | STU3: RequestStatus → R4: RequestStatus (same codes: draft, active, on-hold, revoked, completed, entered-in-error, unknown) |
| `intent` | `intent` | Value set expanded | R4 adds: original-order, reflex-order, filler-order, instance-order, option |
| `priority` | `priority` | None | Same binding: routine, urgent, asap, stat |
| `subject` | `subject` | Reference targets expanded | STU3: Patient, Group → R4: Patient, Group, Location, Device |
| `occurrence[x]` | `occurrence[x]` | Type expanded | STU3: dateTime, Period → R4: dateTime, Period, Timing |
| `authoredOn` | `authoredOn` | None | Same semantics |
| `reasonCode` | `reasonCode` | None | Same type and binding |
| `reasonReference` | `reasonReference` | Reference targets expanded | STU3: Condition, Observation → R4: Condition, Observation, DiagnosticReport, DocumentReference |
| `supportingInfo` | `supportingInfo` | None | Reference(Any) in both |
| `note` | `note` | None | Annotation in both |
| `relevantHistory` | `relevantHistory` | None | Reference(Provenance) in both |

### Elements Renamed or Restructured

| STU3 ReferralRequest | R4 ServiceRequest | Change Description |
|---|---|---|
| `definition` | `instantiatesCanonical` / `instantiatesUri` | Split into two elements. STU3 referenced ActivityDefinition or PlanDefinition directly; R4 separates canonical FHIR references from external URIs |
| `groupIdentifier` | `requisition` | Renamed. Same semantics — a shared identifier linking related requests |
| `type` | `code` | Renamed and broadened. STU3 `type` indicated the type of referral (SNOMED); R4 `code` identifies the specific service being requested (SNOMED procedure codes). Broader in scope since ServiceRequest covers procedures too |
| `serviceRequested` | `category` | Conceptually similar — classifies the type of service. STU3 allowed multiple service codes; R4 `category` classifies the service for searching/sorting. The specific service is now in `code` |
| `context` | `encounter` | Renamed. STU3 allowed Encounter or EpisodeOfCare; R4 allows only Encounter |
| `specialty` | `performerType` | Conceptual shift. STU3 `specialty` indicated the clinical domain (e.g., Cardiology); R4 `performerType` indicates the role/type of desired performer. Similar purpose but different framing |
| `recipient` | `performer` | Renamed and expanded. STU3: Practitioner, Organization, HealthcareService; R4: Practitioner, PractitionerRole, Organization, CareTeam, HealthcareService, Patient, Device, RelatedPerson |
| `requester.agent` + `requester.onBehalfOf` | `requester` | Flattened. STU3 used a BackboneElement with `agent` (who requested) and `onBehalfOf` (the organisation they represented). R4 flattens this to a single Reference, relying on PractitionerRole to convey the organisational context |
| `description` | `note` / `patientInstruction` | STU3's free-text description of the referral has no direct single equivalent. Clinical description maps to `note`; patient-facing instructions map to `patientInstruction` |

### Elements New in R4 (No STU3 Equivalent)

| R4 ServiceRequest Element | Card. | Type | Purpose |
|---|---|---|---|
| `instantiatesUri` | 0..* | uri | External protocol/definition references (separated from canonical) |
| `doNotPerform` | 0..1 | boolean | Indicates the service should NOT be performed (negation pattern) |
| `orderDetail` | 0..* | CodeableConcept | Additional structured order instructions (e.g., catheter type, bandage instructions) |
| `quantity[x]` | 0..1 | Quantity, Ratio, Range | Amount of service being requested |
| `asNeeded[x]` | 0..1 | boolean, CodeableConcept | Preconditions for when the service should occur |
| `locationCode` | 0..* | CodeableConcept | Preferred location (coded) where service should occur |
| `locationReference` | 0..* | Reference(Location) | Preferred location (reference) where service should occur |
| `insurance` | 0..* | Reference(Coverage, ClaimResponse) | Associated insurance/pre-authorisation |
| `specimen` | 0..* | Reference(Specimen) | Relevant specimens for laboratory procedures |
| `bodySite` | 0..* | CodeableConcept | Anatomic location for the procedure |
| `patientInstruction` | 0..1 | string | Patient-facing instructions |

### Elements Removed in R4 (No Direct Equivalent)

| STU3 ReferralRequest Element | Disposition in R4 |
|---|---|
| `description` | No direct equivalent. Use `note` for clinical narrative or `patientInstruction` for patient-facing text |
| `requester.onBehalfOf` | Removed as a separate element. Organisational context is now conveyed through `PractitionerRole` referenced from `requester` |

---

## Status Value Set Comparison

Both versions use the `RequestStatus` value set with the same codes:

| Code | STU3 | R4 | Notes |
|---|---|---|---|
| `draft` | Yes | Yes | Request not yet actionable |
| `active` | Yes | Yes | Request is actionable |
| `on-hold` | Yes | Yes | Temporarily suspended |
| `revoked` | Yes | Yes | Cancelled/withdrawn (was `cancelled` in earlier STU3 drafts) |
| `completed` | Yes | Yes | Activity complete |
| `entered-in-error` | Yes | Yes | Created in error |
| `unknown` | Yes | Yes | Authoring system doesn't know status |

---

## Intent Value Set Comparison

| Code | STU3 | R4 | Notes |
|---|---|---|---|
| `proposal` | Yes | Yes | Suggestion |
| `plan` | Yes | Yes | Intended action |
| `order` | Yes | Yes | Authoritative request |
| `original-order` | No | Yes | New in R4 — first order in a chain |
| `reflex-order` | No | Yes | New in R4 — auto-generated follow-on |
| `filler-order` | No | Yes | New in R4 — order from the fulfiller |
| `instance-order` | No | Yes | New in R4 — specific instance of a recurring order |
| `option` | No | Yes | New in R4 — option within an order set |

---

## Reference Type Changes Summary

| Element | STU3 Reference Targets | R4 Reference Targets |
|---|---|---|
| `basedOn` | ReferralRequest, CarePlan, ProcedureRequest | CarePlan, ServiceRequest, MedicationRequest |
| `replaces` | ReferralRequest | ServiceRequest |
| `subject` | Patient, Group | Patient, Group, Location, Device |
| `context`/`encounter` | Encounter, EpisodeOfCare | Encounter |
| `requester` | Practitioner, Organization, Patient, RelatedPerson, Device (via `agent`) | Practitioner, PractitionerRole, Organization, Patient, RelatedPerson, Device |
| `recipient`/`performer` | Practitioner, Organization, HealthcareService | Practitioner, PractitionerRole, Organization, CareTeam, HealthcareService, Patient, Device, RelatedPerson |
| `reasonReference` | Condition, Observation | Condition, Observation, DiagnosticReport, DocumentReference |

---

## NHS e-RS Extensions on STU3 ReferralRequest

The NHS e-Referral Service FHIR API (STU3) extends the base `ReferralRequest` with several custom extensions to support e-RS-specific business concepts that have no representation in the core FHIR resource. These extensions are defined under the namespace `https://fhir.nhs.uk/STU3/StructureDefinition/`.

The e-RS profile for STU3 is `eRS-ReferralRequest-1`.

### Extension Catalogue

| Extension | URL | Value Type | Purpose |
|---|---|---|---|
| **eRS-Shortlist-SearchCriteria** | `https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-Shortlist-SearchCriteria-1` | Reference (contained) | Links to the contained Parameters resource that captures the search criteria used to build the service shortlist (specialty, clinic type, location, etc.) |
| **eRS-ReferralPriority** | `https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-ReferralPriority-1` | CodeableConcept | The e-RS-specific referral priority (e.g., Routine, Urgent, Two Week Wait) — distinct from the base `priority` element as it uses an e-RS-specific code system |
| **eRS-ReferralState** | `https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-ReferralState-1` | CodeableConcept | The granular e-RS business state of the referral (e.g., AWAITING_TRIAGE, TRIAGED_PROVIDER_RESPONSE, BOOKED, ASSESSMENT_STARTED). More detailed than the base FHIR `status` which only carries draft/active/completed/revoked |
| **eRS-ClinicalInfoFirstSubmitted** | `https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-ClinicalInfoFirstSubmitted-1` | dateTime | Timestamp when clinical referral information (the referral letter) was first submitted by the referrer |
| **eRS-Appointment** | `https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-Appointment-1` | Reference(Appointment) | Links the referral to its associated appointment booking within e-RS |
| **eRS-ReferralShortlist** | `https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-ReferralShortlist-1` | Complex (BackboneElement) | Contains the list of services that the patient was offered to choose from (the "shortlist"), including service IDs and names |
| **eRS-Commissioning-Rule-Organisation** | `https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-Commissioning-Rule-Organisation-1` | Reference(Organization) | Identifies the commissioning organisation associated with the referral for funding/contract routing purposes |
| **eRS-ReferralType** | `https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-ReferralType-1` | CodeableConcept | Distinguishes between referral types within e-RS (e.g., Triage, Appointment, Advice & Guidance) |
| **eRS-ProviderConversionAuthorisation** | `https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-ProviderConversionAuthorisation-1` | CodeableConcept | Indicates whether the provider is authorised to convert an A&G request to a full referral |
| **eRS-ClinicalInfoLastUpdated** | `https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-ClinicalInfoLastUpdated-1` | dateTime | Timestamp when clinical information was last modified |

### Example: e-RS STU3 ReferralRequest with Extensions

```json
{
  "resourceType": "ReferralRequest",
  "id": "000000070000",
  "meta": {
    "profile": [
      "https://fhir.nhs.uk/STU3/StructureDefinition/eRS-ReferralRequest-1"
    ],
    "versionId": "5"
  },
  "extension": [
    {
      "url": "https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-ReferralPriority-1",
      "valueCodeableConcept": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/STU3/CodeSystem/eRS-Priority-1",
            "code": "ROUTINE",
            "display": "Routine"
          }
        ]
      }
    },
    {
      "url": "https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-ReferralState-1",
      "valueCodeableConcept": {
        "coding": [
          {
            "system": "https://fhir.nhs.uk/STU3/CodeSystem/eRS-ReferralState-1",
            "code": "AWAITING_TRIAGE",
            "display": "Awaiting Triage"
          }
        ]
      }
    },
    {
      "url": "https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-ClinicalInfoFirstSubmitted-1",
      "valueDateTime": "2025-03-15T10:30:00+00:00"
    },
    {
      "url": "https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-Shortlist-SearchCriteria-1",
      "valueReference": {
        "reference": "#ServiceSearchCriteria-1"
      }
    },
    {
      "url": "https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-Appointment-1",
      "valueReference": {
        "reference": "Appointment/70000-apt-1"
      }
    },
    {
      "url": "https://fhir.nhs.uk/STU3/StructureDefinition/Extension-eRS-Commissioning-Rule-Organisation-1",
      "valueReference": {
        "reference": "Organization/R69"
      }
    }
  ],
  "status": "active",
  "intent": "order",
  "priority": "routine",
  "serviceRequested": [
    {
      "coding": [
        {
          "system": "https://fhir.nhs.uk/STU3/CodeSystem/eRS-Specialty-1",
          "code": "CARDIOLOGY",
          "display": "Cardiology"
        }
      ]
    }
  ],
  "subject": {
    "identifier": {
      "system": "https://fhir.nhs.uk/Id/nhs-number",
      "value": "9876543210"
    }
  },
  "requester": {
    "agent": {
      "identifier": {
        "system": "https://fhir.nhs.uk/Id/sds-user-id",
        "value": "100102030405"
      }
    },
    "onBehalfOf": {
      "identifier": {
        "system": "https://fhir.nhs.uk/Id/ods-organization-code",
        "value": "A1001"
      }
    }
  },
  "recipient": [
    {
      "identifier": {
        "system": "https://fhir.nhs.uk/Id/ods-organization-code",
        "value": "R69"
      }
    }
  ],
  "authoredOn": "2025-03-15T10:00:00+00:00"
}
```

---

## Migration Considerations: e-RS Extensions in R4

When migrating from STU3 `ReferralRequest` to R4 `ServiceRequest`, the e-RS extensions need to be reassessed. Some can map into core R4 elements; others must remain as extensions (potentially with updated URLs under a new namespace).

| e-RS STU3 Extension | R4 Disposition | Rationale |
|---|---|---|
| **eRS-ReferralPriority** | Core `priority` element + extension for e-RS-specific codes | R4 `priority` carries routine/urgent/asap/stat. Two Week Wait and other NHS-specific priorities still need an extension or a coded `category` |
| **eRS-ReferralState** | Extension (updated namespace) | R4 `status` remains too coarse for e-RS business states. This extension must be retained, mapped to `https://fhir.nhs.uk/StructureDefinition/Extension-eRS-ReferralStatus` |
| **eRS-ClinicalInfoFirstSubmitted** | Extension (retained) | No core R4 element captures this concept. Retain as extension |
| **eRS-ClinicalInfoLastUpdated** | `meta.lastUpdated` or extension | If the concept is "last time clinical info was updated" (distinct from resource update), retain as extension. If it tracks the resource version, `meta.lastUpdated` suffices |
| **eRS-Shortlist-SearchCriteria** | Extension or `supportingInfo` | Could reference a contained Parameters resource via `supportingInfo`. Alternatively retain as extension for clarity |
| **eRS-ReferralShortlist** | `locationReference` + extension | R4 `locationReference` can hold references to shortlisted services/locations. The full shortlist structure (with ordering, names, and selection state) likely still needs an extension |
| **eRS-Appointment** | `supportingInfo` | R4 `supportingInfo` can reference an Appointment resource directly — no extension needed |
| **eRS-Commissioning-Rule-Organisation** | Extension (retained) | No core R4 element for commissioning org. Retain as extension |
| **eRS-ReferralType** | `category` | R4 `category` (0..*) can carry the referral type as an additional classification coding |
| **eRS-ProviderConversionAuthorisation** | Extension (retained) | No core R4 element for this concept |

---

## Structural Comparison Diagram

```
STU3 ReferralRequest                    R4 ServiceRequest
─────────────────────                   ──────────────────
identifier                         ──→  identifier
definition                         ──→  instantiatesCanonical / instantiatesUri
basedOn                            ──→  basedOn
replaces                           ──→  replaces
groupIdentifier                    ──→  requisition
status                             ──→  status
intent                             ──→  intent
type                               ──→  code
priority                           ──→  priority
serviceRequested                   ──→  category
subject                            ──→  subject
context                            ──→  encounter
occurrence[x]                      ──→  occurrence[x]
authoredOn                         ──→  authoredOn
requester.agent                    ──→  requester
requester.onBehalfOf               ──→  (via PractitionerRole)
specialty                          ──→  performerType
recipient                          ──→  performer
reasonCode                         ──→  reasonCode
reasonReference                    ──→  reasonReference
description                        ──→  note / patientInstruction
supportingInfo                     ──→  supportingInfo
note                               ──→  note
relevantHistory                    ──→  relevantHistory
                                   NEW  doNotPerform
                                   NEW  orderDetail
                                   NEW  quantity[x]
                                   NEW  asNeeded[x]
                                   NEW  locationCode
                                   NEW  locationReference
                                   NEW  insurance
                                   NEW  specimen
                                   NEW  bodySite
                                   NEW  patientInstruction
```

---

## Key Migration Risks for e-RS

| # | Risk | Detail | Mitigation |
|---|---|---|---|
| 1 | **Requester structure flattening** | STU3's `requester.agent` + `requester.onBehalfOf` maps to a single `requester` reference in R4. Consumers relying on `onBehalfOf` to identify the referring organisation must adapt | Use `PractitionerRole` with `organization` reference, or carry the organisation as a separate identifier on the requester |
| 2 | **Loss of `description` element** | Free-text referral descriptions have no single home in R4. Must be split between `note` (clinical) and `patientInstruction` (patient-facing) | Define a mapping convention: clinical description → `note[0].text` |
| 3 | **Extension namespace change** | STU3 extensions under `https://fhir.nhs.uk/STU3/StructureDefinition/` must be re-published under an R4 namespace | Publish R4 StructureDefinitions under `https://fhir.nhs.uk/StructureDefinition/` (dropping `/STU3/`) |
| 4 | **Context narrowing** | STU3 `context` allowed EpisodeOfCare; R4 `encounter` does not. Systems using EpisodeOfCare references lose this | Reference EpisodeOfCare via `supportingInfo` or the Encounter's `episodeOfCare` link |
| 5 | **Shortlist representation** | The e-RS shortlist concept (patient choice of services) has no native R4 element. Extension must be retained and potentially restructured | Retain as a custom extension; consider using `locationReference` for basic cases |
| 6 | **Code system alignment** | e-RS uses its own code systems (eRS-Priority, eRS-Specialty, eRS-ReferralState). These must be maintained or mapped to standard terminologies (SNOMED CT) | Dual-code where possible (e-RS code + SNOMED equivalent); publish updated ValueSets |

---

## Summary

The transition from STU3 `ReferralRequest` to R4 `ServiceRequest` is largely a consolidation and broadening exercise. Most STU3 elements have clear R4 equivalents, though several have been renamed or restructured. The key challenges for e-RS are:

1. **Structural changes** — the flattening of `requester`, renaming of `recipient` to `performer`, and loss of `description` require careful mapping
2. **Extension migration** — the majority of e-RS extensions (particularly `ReferralState`, `ClinicalInfoFirstSubmitted`, `CommissioningRuleOrganisation`) must be retained as extensions in R4 since no core elements accommodate these concepts
3. **New opportunities** — R4 elements like `category`, `locationReference`, and `supportingInfo` can absorb some e-RS extension data (referral type, shortlist, appointment linkage), reducing the extension surface area

The R4 `ServiceRequest` is a more capable and flexible resource. For e-RS, the migration path is clear but requires deliberate decisions about which e-RS concepts map to core R4 elements and which must persist as extensions under an updated namespace.

---

## References

- [FHIR STU3 ReferralRequest](https://www.hl7.org/fhir/stu3/referralrequest.html)
- [FHIR R4 ServiceRequest](https://www.hl7.org/fhir/R4/servicerequest.html)
- [e-RS FHIR API](https://digital.nhs.uk/developer/api-catalogue/e-referral-service-fhir)
- [e-RS STU3 Profile: eRS-ReferralRequest-1](https://fhir.nhs.uk/STU3/StructureDefinition/eRS-ReferralRequest-1)
