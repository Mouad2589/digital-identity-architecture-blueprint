# ADR-0004 — eCVRS Integration with the National Identity Platform

## Status

Accepted

## Context

Civil registration — the systematic recording of vital events (birth, death, marriage) — is the legal and administrative foundation of national identity. A birth certificate is a citizen's first identity document. Without it, individuals cannot access healthcare, education, social protection, or obtain a national ID.

In the African context, civil registration completeness is a primary governance challenge:

- Birth registration rates in sub-Saharan Africa average ~46% for children under 5 (UNICEF 2022).
- Without eCVRS integration, identity enrollment depends on citizens presenting pre-existing paper documents, creating a circular dependency: no birth certificate → no ID → no way to prove identity to obtain a birth certificate.
- An integrated eCVRS breaks this cycle by making birth registration the authoritative source of the citizen identifier.

The platform must decide how eCVRS relates to the National Identity Platform: as a module within the same system, or as a peer system with defined integration contracts.

Three options were considered:

**Option A — Single monolith:** eCVRS is a module within the National Identity Platform. Shared database, shared deployment.

**Option B — Peer systems with event bus:** eCVRS and the Identity Platform are independently deployed systems. They communicate via a durable event bus and a shared citizen identifier.

**Option C — eCVRS as read-only data source:** the Identity Platform queries the eCVRS synchronously to validate civil status during enrollment. No event integration.

## Decision

**Option B is adopted.** The eCVRS is deployed as a **peer subsystem** to the National Identity Platform, connected via:

1. **Event bus integration:** vital events (birth, death, marriage) are published to a shared durable event stream. The Identity Platform subscribes and reacts to relevant event types asynchronously.
2. **Shared population identifier:** the `personId` assigned by the eCVRS Population Registry at birth registration is the stable national identifier used across all government systems.
3. **Read API:** the Identity Platform can query the eCVRS Population Registry API to verify civil registration status and retrieve registration data for enrollment.
4. **Independent data stores:** each system maintains its own database. There is no shared schema and no direct cross-system database access.

### Integration contract

| Event / API | Direction | Consumer action |
|---|---|---|
| `BirthRegistered` | eCVRS → Identity Platform | Record birth certificate reference; schedule ID enrollment notification at legal age |
| `DeathRegistered` | eCVRS → Identity Platform | Initiate identity revocation workflow |
| `MarriageRegistered` | eCVRS → Identity Platform | Trigger name change workflow if applicable |
| `GET /population/{personId}` | Identity Platform → eCVRS | Verify civil registration status during enrollment |
| `IdentityRevoked` | Identity Platform → eCVRS | Record identity status in population registry |

### Why Option B over Option A (monolith)

- **Different operational profiles:** eCVRS is optimised for offline-first field operation with eventual consistency; the Identity Platform is optimised for synchronous online verification with strict SLA. Their availability requirements, scaling patterns, and deployment cadences differ fundamentally.
- **Different data lifecycles:** vital records are permanent, immutable legal documents of record; identity records have an active lifecycle (suspend, revoke, renew) subject to data protection rights.
- **Different governance:** civil registration is typically under a different ministry than national identity (Interior vs. Justice vs. Home Affairs). Independent systems support independent governance and budget.
- **Blast radius reduction:** a vulnerability or outage in one system does not cascade to the other. An eCVRS outage does not prevent identity verification.
- **Independent evolution:** eCVRS and Identity Platform can be upgraded, replaced, or procured from different vendors without forcing simultaneous migration.

### Why Option B over Option C (synchronous query only)

- Option C creates a synchronous dependency: identity verification degrades if eCVRS is unavailable.
- Option C misses proactive triggers: the Identity Platform cannot automatically initiate ID enrollment when a child reaches legal age without event consumption.
- Option C misses death-driven revocation: the Identity Platform only learns of deaths when explicitly queried, not at the time of registration.

### Shared identifier assignment

The `personId` is assigned by the eCVRS Population Registry at birth registration. It is:

- A UUID v4, assigned once, never reused, never recycled.
- Included in the birth certificate QR code payload.
- Shared with the Identity Platform via the `BirthRegistered` event.
- Used as the stable citizen identifier across all integrated government systems.

For **adults who were never registered at birth** (a significant population in the target deployment context), the eCVRS generates a `personId` when the late registration is approved by a district supervisor. The Identity Platform then initiates a dedicated late-enrollment workflow.

### Event bus reliability requirements

The event bus between eCVRS and the Identity Platform must provide:

- **At-least-once delivery** with idempotent consumer design.
- **Durable topic retention** of at least 30 days to absorb periods of consumer unavailability.
- **Consumer group isolation:** each downstream system (Identity Platform, Health, Education, etc.) maintains its own consumer group and independent offset.
- A single event bus failure must not block local civil registration operations (eCVRS publishes to the durable topic; downstream systems consume when available).

## Consequences

- **Identity bootstrap chain:** eCVRS becomes the authoritative source for the citizen identifier. Civil registration completeness directly limits identity enrollment coverage. Deployment must prioritise eCVRS rollout in unregistered regions.
- **Event schema governance:** `BirthRegistered`, `DeathRegistered`, and other event schemas must be versioned and backward-compatible. Schema changes require coordination between system teams. A schema registry is required.
- **Late registration path:** adults without a birth registration require a special enrollment workflow with elevated identity proofing. This is a documented exception, not a general bypass of the eCVRS-first principle.
- **Eventual consistency:** downstream systems (Identity Platform, Health, Education) learn of vital events asynchronously. There is a brief window (typically seconds; up to hours if a consumer is recovering) during which systems may have inconsistent views. This is acceptable for civil registration, where real-time propagation is not legally required.
- **Deduplication across systems:** the `personId` is the join key across all government systems. Its uniqueness and integrity must be protected — the eCVRS deduplication service is the gatekeeper.
