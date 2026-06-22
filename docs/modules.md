# Modules

## C4 Container diagram

The container diagram shows the major deployable units of the platform, their responsibilities and their relationships.

```mermaid
flowchart TB
    Citizen(["👤 Citizen"])
    Operator(["👤 Operator"])
    Admin(["👤 Admin"])
    VHW(["👤 Village Health Worker\nField Registrar"])

    subgraph Channels["Channels"]
        Portal["Self-Service Portal\nReact SPA"]
        EnrollStation["Enrollment Station\n.NET App"]
    end

    subgraph Gateway["Gateway"]
        APIGW["API Gateway\nNGINX / Kong"]
        InteropGW["Interoperability GW\nOIDC · SAML · VC"]
    end

    subgraph CoreServices["Core Services"]
        IdentityAPI["Identity API\nASP.NET Core"]
        WorkflowSvc["Workflow Service"]
        IssuanceSvc["Document Issuance"]
        NotifSvc["Notification Service"]
    end

    subgraph DataStores["Data Stores"]
        IdentityDB[("Identity DB\nPostgreSQL")]
        Cache[("Cache\nRedis")]
        AuditStore[("Audit Store\nAppend-only")]
    end

    subgraph Isolated["Isolated Zone"]
        BioVault["Biometric Vault\nASP.NET Core"]
        BioDB[("Biometric DB\nEncrypted")]
    end

    subgraph eCVRS["eCVRS — Civil Registration"]
        FieldApp["Field Registration App\nOffline-first · Android / iOS"]
        SyncEngine["Sync Engine\nBackground Worker"]
        CivEngine["Civil Registration Engine\nEvent Processor"]
        PopReg[("Population Registry\nPostgreSQL")]
    end

    EventBus["Event Bus\nBirthRegistered · DeathRegistered"]

    ObsStack["Observability\nOTel · Grafana · Loki"]
    GovServices["Public Services"]
    SIEM["SIEM / SOC"]
    PrintFacility["Secure Print"]
    eIDAS["eIDAS / EUDI"]

    Citizen --> Portal
    Operator --> EnrollStation
    Admin --> APIGW
    VHW --> FieldApp

    Portal --> APIGW
    EnrollStation --> APIGW
    APIGW --> IdentityAPI
    APIGW --> InteropGW

    IdentityAPI --> IdentityDB
    IdentityAPI --> Cache
    IdentityAPI --> AuditStore
    IdentityAPI --> BioVault
    IdentityAPI --> WorkflowSvc
    IdentityAPI --> ObsStack

    WorkflowSvc --> IssuanceSvc
    WorkflowSvc --> NotifSvc
    BioVault --> BioDB

    IssuanceSvc --> PrintFacility
    InteropGW --> GovServices
    InteropGW --> eIDAS
    AuditStore --> SIEM

    FieldApp --> SyncEngine
    SyncEngine -.->|"Sync when connected"| CivEngine
    CivEngine --> PopReg
    CivEngine --> EventBus
    EventBus -.->|"BirthRegistered / DeathRegistered"| IdentityAPI
```

---

## Module overview

```mermaid
flowchart TB
    subgraph Channels
        Portal[Self-Service Portal]
        EnrollStation[Enrollment Station]
        MobileApp[Mobile App]
        FieldApp[eCVRS Field App\nOffline-first]
    end

    subgraph CivilReg["Civil Registration — eCVRS"]
        CivEngine[Civil Registration Engine]
        PopRegistry[(Population Registry)]
    end

    subgraph Core
        EnrollModule[Enrollment Module]
        IdentityCore[Identity Core]
        BiometricsSvc[Biometric Services]
        WorkflowEngine[Workflow and Case Management]
        AuditModule[Audit and Traceability]
        DocumentIssuance[Document Issuance]
    end

    subgraph Integration
        InteropGW[Interoperability Gateway]
        NotificationSvc[Notification Service]
        EventBus[Event Bus]
    end

    Portal --> IdentityCore
    EnrollStation --> EnrollModule
    MobileApp --> IdentityCore
    EnrollModule --> IdentityCore
    IdentityCore --> BiometricsSvc
    IdentityCore --> WorkflowEngine
    IdentityCore --> AuditModule
    IdentityCore --> DocumentIssuance
    IdentityCore --> InteropGW
    WorkflowEngine --> NotificationSvc

    FieldApp --> CivEngine
    CivEngine --> PopRegistry
    CivEngine --> EventBus
    EventBus --> IdentityCore
```

---

## Enrollment Module

**Responsibility:** Capture, validate and submit identity data for a new enrollment request.

**Key flows:**

- Operator-assisted capture (in-person): biographic data entry + biometric capture (fingerprint, face, iris).
- Online pre-enrollment: citizen submits supporting documents and personal data; operator validates in person.

**Design concerns:**

- Idempotent submission — prevent duplicate enrollment for the same identity.
- Step-based workflow with explicit state machine (started → pending validation → approved / rejected).
- Offline-capable enrollment stations with sync-on-reconnect.

### Enrollment flow — sequence diagram

```mermaid
sequenceDiagram
    actor Operator
    participant Station as Enrollment Station
    participant API as Identity API
    participant Dedup as Biometric Vault
    participant Workflow as Workflow Service
    participant Audit as Audit Store
    participant Notif as Notification Service

    Operator->>Station: Start enrollment session
    Station->>Station: Capture biographic data
    Station->>Station: Capture biometric (fingerprint + face)
    Station->>API: POST /enrollments (biographic + biometric ref)

    API->>API: Validate input (schema + business rules)
    API->>Dedup: Deduplicate(biometricCapture)
    Dedup-->>API: No match found

    API->>API: Create Identity (PendingVerification)
    API->>Audit: Log EnrollmentSubmitted
    API->>Workflow: CreateCase(enrollmentId)
    API-->>Station: 202 Accepted — enrollmentId

    Workflow->>Workflow: Assign case to supervisor
    Workflow->>Notif: Notify supervisor (new case)

    Note over Workflow: Supervisor reviews and approves

    Workflow->>API: ApproveEnrollment(enrollmentId)
    API->>API: Activate Identity
    API->>Audit: Log IdentityActivated
    API->>Notif: Notify citizen (identity ready)
    API-->>Workflow: Confirmed
```

---

## Identity Core

**Responsibility:** Maintain the authoritative identity record, its lifecycle and its relationships.

**Key entities:**

- `Identity` — unique citizen record with stable national identifier.
- `IdentityStatus` — active, suspended, revoked, deceased.
- `BiographicData` — civil registry attributes.
- `BiometricReference` — tokenized reference (never stored in clear).

**Key operations:**

- Create / update / suspend / revoke identity.
- Deduplicate against existing records (biographic + biometric matching).
- Emit domain events on status change (for audit and downstream integration).

---

## Biometric Services

**Responsibility:** Biometric capture quality check, template extraction and 1:1 verification / 1:N deduplication.

**Design concerns:**

- Strict separation: biometric vault is isolated from the rest of the system.
- Biometric data is never returned raw; only match score and decision are exposed.
- SDK integration abstracted behind an interface to allow algorithm provider replacement.

---

## Workflow & Case Management

**Responsibility:** Orchestrate multi-step enrollment and issuance processes with human approval gates.

**Patterns used:**

- State machine per case type (enrollment, renewal, correction, revocation).
- Task assignment to operators with SLA tracking.
- Escalation rules when SLA is breached.

---

## Audit & Traceability

**Responsibility:** Record all sensitive operations in a tamper-evident, queryable audit log.

**Design concerns:**

- Write-only append log; no update or delete operations allowed.
- Each event includes: timestamp, actor, operation, entity reference, outcome, correlation ID.
- Supports legal retention policies and regulatory export.

---

## Document Issuance

**Responsibility:** Generate and deliver identity documents (ID card, passport, digital credential) upon workflow approval.

**Key flows:**

- Physical document: generate personalization data → send to secure print facility → track delivery.
- Digital credential: generate W3C Verifiable Credential or mobile driving licence (ISO 18013-5).

---

## Interoperability Gateway

**Responsibility:** Expose identity verification capabilities to authorized external consumers.

See [interoperability.md](interoperability.md) for details.

---

## eCVRS — Electronic Civil and Vital Registration System

**Responsibility:** Record vital events (birth, death, marriage) and maintain the authoritative population registry that underpins all digital identity services.

**Key flows:**

- Offline birth registration in the field — village health worker on tablet, zero connectivity, instant local confirmation.
- Death registration triggering automatic identity revocation in the National Identity Platform.
- Late registration workflow for previously unregistered adults with elevated supervisor review.

**Design concerns:**

- Offline-first by design — full operation with zero connectivity; sync when connected.
- Immutable, signed event log — each event signed with device PKI key at the moment of capture.
- Eventual consistency — population registry updated asynchronously after sync; no central round-trip required for local confirmation.
- CRDT-based conflict resolution — no silent merges; ambiguous concurrent edits routed to supervisor review queue.
- Offline-verifiable certificates — Ed25519-signed QR codes; key manifest cached on verifier devices.
- Four connectivity tiers explicitly supported: HTTPS sync (Tier 3/2), SMS batch (Tier 1), physical USB/SD transport (Tier 0).

See [ecvrs-module.md](ecvrs-module.md) for full module documentation, [offline-first-design.md](offline-first-design.md) for the sync and resilience architecture, and [ADR-0004](decisions/ADR-0004-ecvrs-civil-registration-integration.md) / [ADR-0005](decisions/ADR-0005-offline-first-architecture-low-connectivity.md) / [ADR-0006](decisions/ADR-0006-crdt-sync-conflict-resolution.md) for the architectural decisions.
