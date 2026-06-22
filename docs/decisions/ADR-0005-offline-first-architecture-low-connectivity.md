# ADR-0005 — Offline-First Architecture for Low-Connectivity Environments

## Status

Accepted

## Context

The digital identity and civil registration systems are deployed across urban, semi-urban, and rural areas. Field registration workers (village health workers, district civil registrars) operate in environments where:

- Internet connectivity is absent for extended periods — hours to weeks.
- SMS (2G) is often the only available communication channel.
- Power supply is unreliable; field devices operate on battery charged by solar panels.
- Physical transport of data (USB, SD card) is a legitimate and common operational procedure.
- NTP synchronisation may be unavailable, causing device clocks to drift.

A system designed on the assumption of reliable connectivity will fail in these conditions, and the citizens in the most underserved areas — who need civil registration services most — will be systematically excluded.

Two architectural approaches were considered:

**Option A — Online-first with offline degradation:** the system is designed for online operation; offline mode is a fallback that permits limited read-only use or caches a subset of data. Core write operations require connectivity.

**Option B — Offline-first with online synchronisation:** the system is designed for offline operation as the primary mode; connectivity enhances speed and freshness but is never a prerequisite for any core operation.

## Decision

**Option B is adopted.** Every field-facing component is designed to operate in full offline mode. Connectivity is an optimisation that reduces sync latency; it is not a dependency.

### Architectural constraints derived from this decision

**1. Local event store is mandatory.**

All events are persisted locally (SQLite WAL) before any network operation is attempted. The user always receives a local confirmation. No event is ever dependent on a successful network call to be durably captured.

**2. Events are immutable and signed at capture.**

Every event is signed with the field device's private key (stored in device TEE) at the moment of capture. Signatures allow the central system to detect post-capture tampering. Events cannot be modified after signing; corrections create new amendment events.

**3. Sync is asynchronous and non-blocking.**

Sync never blocks the registration UI. The registrar can continue working while sync runs in the background. The UI shows a transparent sync status indicator (pending / uploaded / acknowledged).

**4. All processing is idempotent at every layer.**

Events carry a client-generated UUID (`event_id`). Re-delivery of the same event due to retry after network failure is a no-op at the server. No event is processed twice. No unique-constraint violation is surfaced to the user for re-synced events.

**5. Offline-verifiable outputs.**

Certificates and receipts issued by the system are verifiable without connectivity, using Ed25519 cryptographic signatures and cached public key manifests. The verifier never requires a live API call to validate a certificate.

**6. District hubs are independent operational nodes.**

The district hub (replication relay) operates fully when the national layer is unreachable. Field devices sync to the district hub; the hub batches and forwards to the national layer. A national outage does not prevent district-level or field-level operation.

**7. No operation requires central confirmation to complete locally.**

No workflow step for the registrar is blocked waiting for a server response. Central validation (deduplication, duplicate detection, population registry update) happens asynchronously after sync. The registrar's locally confirmed event is legally valid from the moment of signing.

**8. Four connectivity tiers are explicitly supported.**

| Tier | Transport | Sync mechanism |
|---|---|---|
| Tier 3 — Connected | HTTPS | Real-time batch sync |
| Tier 2 — Intermittent | HTTPS | Background sync with exponential backoff |
| Tier 1 — SMS only | SMS modem | Compact SMS batch protocol (minimum vital fields) |
| Tier 0 — Isolated | Physical device | USB or SD card transport to district vehicle |

### What is deferred to central processing (not required offline)

The following capabilities require connectivity and run asynchronously after sync:

- Population deduplication (1:N matching against the full registry).
- Formal certificate generation (print-quality PDF with security features).
- Downstream system notification (Health, Education, Electoral, Identity Platform).
- Population registry update and statistics aggregation.
- Supervisor review queue for validation failures.

### What is available fully offline

- Complete data capture for all vital event types.
- Local schema validation and business rule checks.
- Immediate interim receipt generation (simplified A4 or thermal print).
- Ed25519-signed QR code generation for interim receipt (offline-verifiable).
- Sync status dashboard (pending / uploaded / acknowledged counts).
- Draft form recovery after power loss.

## Consequences

- **Eventual consistency:** the population registry reflects registered events with a latency proportional to the sync tier of the originating device. Real-time accuracy is traded for availability and coverage. This is acceptable for civil registration, where timeliness is measured in days, not milliseconds.
- **Deduplication delay:** duplicate detection runs after sync. A duplicate registration may exist locally for hours or days before being flagged. The supervisor review queue is the operational safety net.
- **Device management complexity:** each field device requires a PKI certificate, a provisioning ceremony, and a decommissioning process. Lost or stolen devices must be deprovisioned promptly to prevent fraudulent event injection. A device registry and revocation infrastructure are required.
- **Registrar training:** field registrars must understand sync status indicators and respond appropriately when events are stale (e.g., locate connectivity or arrange physical transport). UX must make sync status unambiguous and non-technical.
- **Testing infrastructure:** offline scenarios require deliberate test infrastructure: network simulation, chaos engineering, clock skew injection. Standard CI pipelines must include offline scenario test suites. See [offline-first-design.md](../offline-first-design.md) for the full test catalogue.
- **Audit completeness guarantee:** every event captured offline must eventually reach the central audit log. The system must guarantee no event is permanently lost. Physical transport (Tier 0) is the guaranteed backstop for Tier 0 devices.
- **Clock reliability:** HLC timestamps (see [ADR-0006](ADR-0006-crdt-sync-conflict-resolution.md)) provide causal ordering without relying on NTP. Wall-time accuracy is best-effort; causal correctness is guaranteed.
