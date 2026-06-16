# Replacing the e-RS HL7 V3 API with BaRS-Aligned Operations

## Purpose

This document describes how the BaRS-aligned `/ServiceRequest` API (via the [translation layer](./ers-to-servicerequest-translation-layer.md)) combined with the [BaRS Appointments Standard Pattern](../BaRS-Appointments-StandardPattern/README.md) can fully replace the [e-RS HL7 V3 API](https://digital.nhs.uk/developer/api-catalogue/e-referral-service-hl7-v3).

The e-RS HL7 V3 API is a legacy interface used primarily by secondary care Patient Administration Systems (PAS) to:

1. **Publish appointment slots** from PAS into e-RS (so referrers can book)
2. **Poll for referrals** assigned to their services
3. **Create paperless referrals** into secondary care

All of these capabilities are replaced by standard FHIR R4 operations under the BaRS model.

---

## Background: What Is the e-RS HL7 V3 API?

The e-RS HL7 V3 API (historically the "Choose and Book" domain) is a Spine-connected interface that uses:

- **HL7 V3 SOAP web services** for synchronous interactions
- **HL7 V3 ebXML messaging** for asynchronous interactions
- **TLS-MA (mutual TLS) authentication** — application-restricted, no end-user identity
- **HSCN network connectivity** — not internet-facing

### Key Interactions (from Spine MIM — Choose and Book Domain)

| # | HL7 V3 Interaction | Direction | Purpose |
|---|---|---|---|
| 1 | **Publish Appointment Slots** | PAS → Spine → e-RS | Sends available appointment slots from secondary care PAS to e-RS, making them bookable by referrers |
| 2 | **Query Appointment Slot Availability** | e-RS → PAS | e-RS queries PAS for real-time slot availability when a referrer searches |
| 3 | **Book Appointment** | e-RS → PAS | When a referrer books via e-RS, the booking is pushed to the PAS |
| 4 | **Cancel/Rebook Appointment** | e-RS → PAS | Appointment changes pushed to PAS |
| 5 | **Accept Referral** | PAS → e-RS | PAS acknowledges receipt of referral |
| 6 | **Retrieve Referral Details** | PAS → e-RS | PAS pulls full referral details after receiving notification |
| 7 | **Update Referral Status** | PAS → e-RS | PAS reports status changes (attended, DNA, discharged) back to e-RS |

### Current Architecture (HL7 V3)

```
┌────────────────┐       HL7 V3 SOAP / ebXML        ┌────────────────┐
│                │◄─────────────────────────────────▶│                │
│  Secondary     │         (via NHS Spine)            │     e-RS       │
│  Care PAS      │                                    │   (Central)    │
│                │   • Publish slots (async)           │                │
│  (e.g. Cerner, │   • Query slots (sync)             │                │
│   Epic, etc.)  │   • Book/Cancel (sync)             │                │
│                │   • Retrieve referral (sync)        │                │
└────────────────┘   • Update status (async)          └────────────────┘
        │                                                      │
        │ HSCN Network                                         │
        │ TLS-MA Auth                                          │
        │ MHS Adaptor (ebXML)                                  │
```

### Why It Needs Replacing

| Problem | Detail |
|---|---|
| **Legacy technology** | HL7 V3 SOAP/ebXML is complex, expensive to maintain, and has a shrinking talent pool |
| **Spine coupling** | Requires HSCN connectivity and Spine message routing — heavy infrastructure |
| **No FHIR alignment** | Uses HL7 V3 data model, not interoperable with modern FHIR R4 systems |
| **MHS Adaptor dependency** | Consumers must run or integrate with the MHS Adaptor for ebXML messaging |
| **Limited to e-RS** | Tightly coupled to the e-RS monolith — cannot be reused for other booking/referral patterns |
| **Onboarding complexity** | Requires Common Assurance Process (CAP), HSCN, endpoint registration on Spine |

---

## Target Architecture: BaRS-Aligned Replacement

```
┌────────────────┐      FHIR R4 (HTTPS/REST)       ┌────────────────┐
│                │◄───────────────────────────────▶│                │
│  Secondary     │                                  │  BaRS Proxy    │
│  Care PAS      │   Standard Pattern:              │  (Transport)   │
│                │   • GET /Slot                     │                │
│  (e.g. Cerner, │   • POST /Appointment            └───────┬────────┘
│   Epic, etc.)  │   • PUT /Appointment                     │
│                │   • GET /ServiceRequest                   │
│                │   • PATCH /ServiceRequest                 ▼
└────────────────┘                              ┌────────────────────┐
        │                                       │  Translation Layer │
        │ Internet (TLS)                        │  (e-RS Facade)     │
        │ App-restricted JWT                    │                    │
        │ No MHS Adaptor needed                 └────────────────────┘
```

### Key Changes

| Aspect | HL7 V3 (Current) | BaRS (Target) |
|---|---|---|
| **Protocol** | HL7 V3 SOAP + ebXML | FHIR R4 RESTful (HTTPS) |
| **Network** | HSCN only | Internet (TLS) — no HSCN connection to BaRS |
| **Auth** | TLS-MA (certificate-based) | Signed JWT (app-restricted) or CIS2 |
| **Messaging** | Async ebXML via MHS | Synchronous REST (fire-and-forget for events via MNS) |
| **Data model** | HL7 V3 CDA / RIM | FHIR R4 (UKCore profiles) |
| **Infrastructure** | MHS Adaptor + Spine endpoint registration | Standard HTTPS client; no special adaptor |
| **Slot management** | PAS pushes/publishes slots to e-RS centrally | PAS **is** the slot authority — BaRS queries PAS directly via `GET /Slot` |
| **Coupling** | Tightly coupled to e-RS monolith | Loosely coupled; PAS is a BaRS Receiver, translation layer serves legacy data |

---

## Interaction-by-Interaction Replacement

### 1. Publish Appointment Slots

**HL7 V3 (Current):**
- PAS publishes slots to e-RS via async ebXML message
- e-RS stores the slots centrally
- When a referrer searches, e-RS serves slots from its own store (or queries PAS in real-time)

**BaRS (Target):**
- PAS no longer publishes slots to a central store
- PAS **is the slot authority** — it holds slots locally and responds to `GET /Slot` queries directly
- The referrer (or BaRS Proxy on behalf of the referrer) queries the PAS in real-time

```
HL7 V3:  PAS ──publish slots──▶ e-RS (stores centrally) ──serve──▶ Referrer

BaRS:    Referrer ──GET /Slot──▶ BaRS Proxy ──GET /Slot──▶ PAS (responds directly)
```

**What the PAS must implement:**

| Operation | Detail |
|---|---|
| `GET /Slot` | Respond to slot search queries filtered by `Schedule.actor:HealthcareService`, `start`, `end`, `status` |
| `GET /metadata` | Return a CapabilityStatement declaring support for Slot search and Appointment create |

**Benefits:**
- Eliminates the "slot publishing" batch/async process entirely
- Slots are always real-time (no stale data from publishing lag)
- No central slot store to maintain
- PAS retains full control over its own schedule data

**Migration consideration:**
During transition, the translation layer can also serve a `GET /Slot` facade over ersdb's stored slot data for PAS systems not yet capable of responding directly. This provides continuity while PAS vendors build their FHIR Receiver capability.

---

### 2. Query Appointment Slot Availability

**HL7 V3 (Current):**
- e-RS queries PAS via synchronous HL7 V3 SOAP to check real-time availability
- Or serves from its centrally-stored published slots

**BaRS (Target):**
- Replaced entirely by `GET /Slot` (see above)
- Standard FHIR search parameters: `Schedule.actor:HealthcareService`, `start`, `end`, `status`

**Example:**

```http
GET /Slot?Schedule.actor:HealthcareService=2000072489
    &start=ge2025-07-01T00:00:00+00:00
    &end=le2025-07-01T23:59:59+00:00
    &status=free HTTP/1.1
Host: int.api.service.nhs.uk
Authorization: Bearer {jwt}
NHSD-Target-Identifier: {base64-encoded service identifier}
```

---

### 3. Book Appointment

**HL7 V3 (Current):**
- When a referrer books via e-RS, the booking is pushed to the PAS via HL7 V3 SOAP
- The PAS must implement a Spine endpoint to receive the booking message

**BaRS (Target):**
- The referrer (or any sender) creates the booking directly on the PAS via `POST /Appointment`
- The PAS is a BaRS Receiver — it accepts Appointment resources

**What the PAS must implement:**

| Operation | Behaviour |
|---|---|
| `POST /Appointment` | Accept a booking, allocate the slot, return the created Appointment with server-assigned ID |
| Validation | Confirm the referenced Slot is still free; reject with 409 if already booked |

**Linking to a Referral:**
In the BaRS model, an Appointment can reference the ServiceRequest it relates to:

```json
{
  "resourceType": "Appointment",
  "status": "booked",
  "slot": [{"reference": "Slot/deb4c4b3-870b-4599-84df-5e54cef7afda"}],
  "basedOn": [{"reference": "ServiceRequest/000000070000"}],
  "participant": [...]
}
```

The `basedOn` field links the appointment to the referral (ServiceRequest) — replacing the implicit UBRN-to-appointment association in e-RS.

---

### 4. Cancel / Rebook Appointment

**HL7 V3 (Current):**
- e-RS pushes cancellation/rebooking to PAS via HL7 V3 message
- PAS updates its local diary

**BaRS (Target):**
- Cancel: `PUT /Appointment/{id}` with `status: cancelled`
- Rebook: Cancel the existing appointment, search for new slot, create new appointment

See: [Cancel Appointment](../BaRS-Appointments-StandardPattern/05-cancel-appointment.md) and [Rebook Appointment](../BaRS-Appointments-StandardPattern/07-rebook-appointment.md).

---

### 5. Accept Referral / Retrieve Referral Details

**HL7 V3 (Current):**
- PAS receives notification of a new referral via ebXML
- PAS calls back to e-RS via HL7 V3 SOAP to retrieve full referral details (clinical info, attachments, patient demographics)
- PAS sends an acceptance acknowledgement

**BaRS (Target):**
- PAS queries for referrals assigned to its service via `GET /ServiceRequest?performer:identifier={ods}`
- Full referral details are returned inline in the Bundle (no separate retrieval step)
- Acceptance is recorded via `PATCH /ServiceRequest/{id}` (status progression)

**Via the Translation Layer:**

```http
GET /ServiceRequest?performer:identifier=https://fhir.nhs.uk/Id/ods-organization-code|RXF
    &status=active HTTP/1.1
```

Returns all active referrals for the PAS's organisation as FHIR R4 ServiceRequest resources — the translation layer handles the ersdb query and STU3→R4 mapping transparently.

**Accepting a referral:**

```http
PATCH /ServiceRequest/000000070000 HTTP/1.1
Content-Type: application/json-patch+json

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

---

### 6. Update Referral Status

**HL7 V3 (Current):**
- PAS sends status updates back to e-RS via async ebXML messages (attended, DNA, discharged, etc.)
- e-RS updates its central referral record

**BaRS (Target):**
- PAS updates the ServiceRequest status via `PATCH /ServiceRequest/{id}`
- The translation layer writes through to ersdb, maintaining e-RS as system of record during transition

**Common status updates from PAS:**

| Clinical Event | PATCH Operation | e-RS Status (extension) | R4 Status |
|---|---|---|---|
| Patient attended | Replace status | `ASSESSMENT_STARTED` | `active` |
| Patient DNA | Replace status | `DID_NOT_ATTEND` | `active` |
| Treatment complete | Replace status | `COMPLETED` | `completed` |
| Patient discharged | Replace status | `DISCHARGED` | `completed` |
| Referral rejected at triage | Replace status | `REJECTED` | `revoked` |

---

### 7. Create Paperless Referral (Primary → Secondary)

**HL7 V3 (Current):**
- GP system creates a referral via HL7 V3 (deprecated — already replaced by e-RS FHIR API for most use cases)

**BaRS (Target):**
- Already fully handled by `POST /ServiceRequest` via BaRS
- The translation layer writes through to ersdb for referrals still managed by the e-RS backend
- New referrals created post-cutover go to the new Referral Service directly

> **Note:** The HL7 V3 interaction for referral creation from primary care is already deprecated and replaced by the [e-RS FHIR API](https://digital.nhs.uk/developer/api-catalogue/e-referral-service-fhir). The BaRS `POST /ServiceRequest` replaces both.

---

## Summary: Full Interaction Mapping

| # | HL7 V3 Interaction | BaRS Replacement | Resource | Direction |
|---|---|---|---|---|
| 1 | Publish Appointment Slots | **Eliminated** — PAS responds to `GET /Slot` directly | `Slot` | Consumer → PAS |
| 2 | Query Slot Availability | `GET /Slot` | `Slot`, `Schedule` | Consumer → PAS |
| 3 | Book Appointment | `POST /Appointment` | `Appointment` | Consumer → PAS |
| 4 | Cancel Appointment | `PUT /Appointment/{id}` (status=cancelled) | `Appointment` | Consumer → PAS |
| 5 | Accept Referral | `PATCH /ServiceRequest/{id}` | `ServiceRequest` | PAS → Translation Layer |
| 6 | Retrieve Referral Details | `GET /ServiceRequest?performer:identifier={ods}` | `ServiceRequest` (Bundle) | PAS → Translation Layer |
| 7 | Update Referral Status | `PATCH /ServiceRequest/{id}` | `ServiceRequest` | PAS → Translation Layer |
| 8 | Create Referral (deprecated) | `POST /ServiceRequest` | `ServiceRequest` | GP → BaRS |

---

## What PAS Vendors Must Implement

### As a BaRS Receiver (Slot & Appointment Management)

The PAS takes on the role of a **BaRS Receiver** for slot and appointment operations. This means the PAS exposes a FHIR R4 endpoint that BaRS (or consumers directly) can call.

| Capability | Endpoint | Description |
|---|---|---|
| **Capability Statement** | `GET /metadata` | Declare supported operations |
| **Slot Search** | `GET /Slot` | Return available slots for a given service/schedule within a date range |
| **Create Appointment** | `POST /Appointment` | Accept a booking, allocate the slot |
| **Read Appointment** | `GET /Appointment/{id}` | Return a previously booked appointment |
| **Update Appointment** | `PUT /Appointment/{id}` | Accept status changes (cancelled, etc.) |

### As a BaRS Consumer (Referral Retrieval & Status Updates)

The PAS also acts as a **BaRS Consumer** when it needs to pull referrals and push status updates. It calls the translation layer (via BaRS Proxy) to access referral data.

| Capability | Endpoint | Description |
|---|---|---|
| **Search Referrals** | `GET /ServiceRequest?performer:identifier={ods}` | Retrieve referrals assigned to this org |
| **Read Referral** | `GET /ServiceRequest/{id}` | Get full details of a specific referral |
| **Update Referral** | `PATCH /ServiceRequest/{id}` | Report status changes (accepted, DNA, completed) |

---

## Slot Management: The Paradigm Shift

The most significant architectural change is in **slot management**. The HL7 V3 model requires PAS to actively push slots to e-RS. The BaRS model inverts this:

| Aspect | HL7 V3 (Push) | BaRS (Pull) |
|---|---|---|
| **Slot authority** | e-RS (central store) | PAS (local, authoritative) |
| **Data flow** | PAS → e-RS (publish batch) | Consumer → PAS (query on demand) |
| **Freshness** | Stale (published periodically) | Real-time (queried live) |
| **PAS responsibility** | Format and push HL7 V3 messages | Respond to `GET /Slot` FHIR queries |
| **Failure mode** | Slots disappear if publish fails | Consumer retries the query |
| **Slot issues** | Central "no slots available" problem when PAS fails to publish | Only occurs if PAS is genuinely full or unreachable |

### Eliminating the "Slot Availability Issue"

A well-documented problem with e-RS is the "appointment slot issue" — where services appear to have no available slots because:

- The PAS failed to publish slots (batch job failure)
- The publishing lag means newly-created slots aren't yet visible
- Slot ranges in e-RS don't match the PAS polling configuration

Under the BaRS model, this entire category of problem is eliminated. Slots are queried directly from the PAS in real-time. If the PAS has free slots, they're visible immediately. If it doesn't, the "no slots" response is genuine, not a synchronisation artefact.

---

## Migration Path for PAS Vendors

### Phase 1: Translation Layer as Intermediary

During the initial transition, PAS vendors continue to operate as before (HL7 V3 or via e-RS FHIR STU3 API). The translation layer provides the FHIR R4 interface externally while interacting with ersdb internally.

```
Referrer ──GET /Slot──▶ Translation Layer ──query──▶ ersdb (slot table)
                                                      ↑
                                          PAS still publishes slots here via HL7 V3
```

This gives PAS vendors time to build their BaRS Receiver capability without breaking existing flows.

### Phase 2: PAS Implements BaRS Receiver (Slot & Appointment)

PAS vendors implement:
- `GET /Slot` — serving slots from their own schedule database
- `POST /Appointment` — accepting bookings directly
- `GET /metadata` — advertising their capabilities

At this point, the slot management flow bypasses the translation layer entirely:

```
Referrer ──GET /Slot──▶ BaRS Proxy ──GET /Slot──▶ PAS (responds directly, real-time)
```

### Phase 3: PAS Implements BaRS Consumer (Referral Operations)

PAS vendors switch from HL7 V3 referral retrieval/status updates to:
- `GET /ServiceRequest` — pulling referrals from the translation layer
- `PATCH /ServiceRequest` — pushing status updates via REST

### Phase 4: HL7 V3 Decommission

Once all connected PAS systems have migrated:
- Disable HL7 V3 Spine endpoint for slot publishing
- Remove ebXML messaging infrastructure
- Decommission MHS Adaptor requirement for PAS
- Retire the slot publishing batch process

### Timeline Considerations

| Phase | Dependency | Estimated Duration |
|---|---|---|
| Phase 1 (Translation Layer) | Translation layer build + test | 6–12 months |
| Phase 2 (PAS as Receiver) | PAS vendor development cycles | 12–24 months per vendor |
| Phase 3 (PAS as Consumer) | Referral operations via REST | Parallel with Phase 2 |
| Phase 4 (HL7 V3 decommission) | All PAS vendors migrated | 3–5 years from start |

> **Reality check:** The HL7 V3 API cannot be switched off until *all* connected PAS systems have migrated. Given the diversity of the PAS landscape (Cerner, Epic, System C, Meditech, plus bespoke systems), this is a multi-year programme. The translation layer ensures the BaRS interface is available immediately while the long tail of PAS vendors catches up.

---

## Network and Connectivity Changes

| Aspect | HL7 V3 (Current) | BaRS (Target) |
|---|---|---|
| **Network** | HSCN (private NHS network) | Internet (public, TLS-secured) — no HSCN |
| **Connectivity** | Point-to-point via Spine routing | HTTPS to BaRS Proxy (or direct to PAS endpoint) |
| **Adaptor** | MHS Adaptor required for ebXML | No adaptor — standard HTTPS client |
| **Endpoint registration** | Spine endpoint (SDS/LDAP) | Endpoint Catalogue (or direct URL config) |
| **Firewall rules** | HSCN peering + Spine allow-listing | Standard HTTPS egress (port 443) — public internet |

This dramatically lowers the barrier to entry. Any system that can make HTTPS calls with a JWT can participate — no HSCN, no MHS, no Spine endpoint registration needed. BaRS is internet-only; there is no HSCN connectivity option.

---

## Authentication and Security Comparison

| Aspect | HL7 V3 | BaRS |
|---|---|---|
| **Authentication** | TLS-MA (client certificate on HSCN) | Signed JWT (RFC 7523) — app-restricted, internet-facing |
| **Identity** | Certificate CN identifies the system | JWT `iss` claim identifies the application |
| **User context** | Not required (application-restricted) | Not required (app-restricted) — CIS2 optional for user-restricted flows |
| **Authorisation** | Implicit via Spine RBAC + endpoint registration | JWT claims + `NHSD-End-User-Organisation` header validated against Endpoint Catalogue |
| **Token lifetime** | N/A (certificate-based) | 5 minutes (short-lived JWT) |
| **Key management** | X.509 certificate renewal (annual) | JWKS key pair (rotatable) |

The BaRS model is simpler to implement (no PKI infrastructure needed) and more aligned with modern API security patterns (OAuth2 / JWT).

---

## What the Translation Layer Provides During Transition

During the migration period, the translation layer serves a dual role:

### For BaRS consumers (referrers, new systems):
- `GET /ServiceRequest` — retrieves referral data from ersdb as FHIR R4
- `POST /ServiceRequest` — creates new referrals (writes through to ersdb)
- `GET /Slot` — serves slots from ersdb's slot store (while PAS still publishes via HL7 V3)

### For legacy PAS systems (still on HL7 V3):
- Their existing HL7 V3 integration continues to work unchanged
- Slots they publish via HL7 V3 are visible through the translation layer's `GET /Slot`
- Referrals assigned to them are accessible via both HL7 V3 (as before) and the new R4 interface

This means a new BaRS-capable system can interact with a legacy HL7 V3 PAS without either side needing to change — the translation layer bridges the gap.

---

## Event Notifications: Replacing ebXML Async Messages

The HL7 V3 API uses ebXML for asynchronous notifications (e.g., "a new referral has been assigned to your service"). In the BaRS model, this is replaced by:

### Option A: Polling with Version IDs (Near-term)

PAS systems poll `GET /ServiceRequest?performer:identifier={ods}&status=active` periodically. Each ServiceRequest includes `meta.versionId` — the PAS compares against its local store and only processes referrals whose version has changed.

This is functionally equivalent to the current HL7 V3 polling pattern but uses standard REST and is far simpler to implement.

### Option B: Multicast Notification Service (Strategic)

The [Multicast Notification Service (MNS)](https://digital.nhs.uk/developer/api-catalogue/multicast-notification-service) provides push-based event notifications. When a referral is created or updated, an event is published:

```
ServiceRequest created/updated → MNS → Subscribed PAS systems notified
```

The PAS subscribes to events for its organisation and receives a lightweight notification (containing the ServiceRequest ID and change type). It then calls `GET /ServiceRequest/{id}` to retrieve the full details.

This eliminates polling entirely and provides near-real-time notification — a significant improvement over both the HL7 V3 ebXML model and REST polling.

### Comparison

| Aspect | HL7 V3 ebXML | REST Polling | MNS (Pub/Sub) |
|---|---|---|---|
| **Latency** | Seconds–minutes (Spine queue) | Poll interval (e.g., 5 min) | Near-real-time (seconds) |
| **Infrastructure** | MHS Adaptor + Spine endpoint | Standard HTTPS client | HTTPS + MNS subscription |
| **Complexity** | High (ebXML, reliable messaging) | Low (simple GET + compare) | Medium (subscription management) |
| **Missed messages** | Possible (delivery failures) | No (polling is idempotent) | No (catch-up mechanism) |
| **Load** | Low (push-based) | Higher (repeated polling) | Low (push-based) |

---

## Impact on the PAS Vendor Ecosystem

### Current PAS Vendor Integration Landscape

Major PAS vendors with existing HL7 V3 integrations to e-RS:

| Vendor | System | HL7 V3 Integration | FHIR Capability |
|---|---|---|---|
| Oracle Health | Cerner Millennium | Yes (mature) | FHIR R4 server available |
| Epic | Epic | Yes (via bridges) | FHIR R4 server available |
| Dedalus | System C (Medway) | Yes | FHIR capability in development |
| Meditech | Meditech Expanse | Yes | FHIR R4 server available |
| Various | Bespoke/legacy PAS | Yes (via adaptors) | Varies — many have no FHIR capability |

### Vendor Readiness Assessment

| Readiness Level | Description | Vendors | Migration Effort |
|---|---|---|---|
| **Ready** | Already has FHIR R4 server; minimal config to enable BaRS Receiver | Epic, Cerner, Meditech | Low — configure existing FHIR server to support `GET /Slot` and `POST /Appointment` |
| **Capable** | Has FHIR development capability; needs to build BaRS endpoints | System C, newer systems | Medium — build and test BaRS Receiver endpoints |
| **Legacy** | No FHIR capability; will need adaptor or replacement | Bespoke/very old PAS | High — requires intermediary adaptor or system replacement |

### For Legacy PAS Without FHIR Capability

Systems that cannot implement a FHIR R4 Receiver can use a **PAS-side adaptor** — a lightweight facade that:

1. Exposes `GET /Slot` and `POST /Appointment` to BaRS consumers
2. Translates FHIR requests into the PAS's native API or database
3. Returns FHIR responses

This is the inverse of the e-RS translation layer — it sits in front of the PAS rather than in front of the database:

```
BaRS Proxy ──GET /Slot──▶ PAS Adaptor (FHIR → native) ──▶ Legacy PAS
```

---

## Onboarding Comparison

| Step | HL7 V3 (Current) | BaRS (Target) |
|---|---|---|
| **Assurance** | Common Assurance Process (CAP) — tailored for e-RS, heavy documentation | BaRS onboarding — lighter, self-service where possible |
| **Network** | HSCN connection required | Internet only (no HSCN) |
| **Endpoint registration** | Spine Directory Service (SDS/LDAP) | Endpoint Catalogue registration |
| **Testing** | Path to Live environments (PTL) | BaRS INT environment |
| **Certificate** | PKCS#12 client certificate for TLS-MA | API key + JWKS key pair for signed JWT |
| **MHS** | Deploy and configure MHS Adaptor | Not required |
| **Timeline** | Months (network + assurance + certificate) | Weeks (API key + Endpoint Catalogue + testing) |

---

## Risks and Mitigations

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| 1 | PAS vendors slow to adopt FHIR Receiver capability | Prolongs HL7 V3 dependency | Translation layer serves as bridge; run both in parallel |
| 2 | Real-time `GET /Slot` creates load on PAS systems not designed for high query rates | PAS performance issues | Rate limiting, caching guidance, capacity planning with PAS vendors |
| 3 | Loss of central slot store removes e-RS visibility into slot utilisation metrics | Reporting/analytics gap | Require PAS to report slot utilisation via events (MNS) or periodic aggregation |
| 4 | Transition period requires supporting both HL7 V3 and FHIR R4 simultaneously | Operational complexity | Clear cutover criteria per PAS vendor; sunset timeline |
| 5 | PAS-side adaptors for legacy systems add a new maintenance burden | Cost, fragility | Provide reference implementation; encourage system modernisation over adaptation |
| 6 | Different PAS vendors may implement `GET /Slot` with inconsistent behaviour | Interoperability failures | Publish conformance test suite; mandate BaRS compliance testing |
| 7 | Real-time slot queries fail if PAS is unreachable | Booking not possible | Fallback: show "service unavailable" (honest signal vs. stale published slots) |

---

## Summary

The e-RS HL7 V3 API can be fully replaced by a combination of:

1. **BaRS Appointments Standard Pattern** (`GET /Slot`, `POST /Appointment`, `PUT /Appointment`) — replacing slot publishing and appointment booking/management
2. **BaRS-aligned `/ServiceRequest` operations via the Translation Layer** (`GET /ServiceRequest`, `PATCH /ServiceRequest`) — replacing referral retrieval and status updates
3. **Multicast Notification Service** — replacing ebXML async notifications

The key architectural shift is:
- **Slots**: From "PAS pushes to central store" → "PAS is queried directly in real-time"
- **Referrals**: From "e-RS pushes to PAS via ebXML" → "PAS pulls from translation layer via REST"
- **Status**: From "PAS pushes to e-RS via ebXML" → "PAS updates via REST PATCH"

The translation layer enables this transition without requiring all PAS vendors to migrate simultaneously. New BaRS-capable consumers get a modern FHIR R4 interface immediately, while legacy PAS systems continue operating via HL7 V3 until they migrate. The long-term end state is the complete decommission of the HL7 V3 API, the MHS Adaptor dependency, and the HSCN-only connectivity requirement.
