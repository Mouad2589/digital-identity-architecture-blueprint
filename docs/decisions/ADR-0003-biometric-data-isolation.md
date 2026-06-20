# ADR-0003 — Biometric Data Isolation Architecture

## Status

Accepted

## Context

Biometric data (fingerprints, facial images, iris scans) is among the most sensitive personal data that exists. Unlike passwords, biometric characteristics cannot be changed if compromised. This creates irreversible privacy harm.

Several regulatory and security requirements apply:
- GDPR Article 9 classifies biometric data as **special category data** requiring explicit legal basis and elevated protection.
- The risk of a breach is existential to the platform's trust — a biometric database breach cannot be mitigated by resetting credentials.
- Biometric matching algorithms must be upgradeable without migrating the entire identity database.

The naive approach — storing biometric templates alongside identity records in the main database — creates an unacceptable blast radius in case of a breach.

## Decision

Biometric data is stored and processed in a **dedicated, network-isolated Biometric Vault**, separate from all other platform components.

### Architectural constraints

1. **No raw biometric data leaves the vault.** The vault API exposes only:
   - `Enroll(subjectId, biometricCapture) → templateId` — stores a template, returns an opaque reference.
   - `Verify(subjectId, biometricCapture) → MatchResult` — 1:1 verification; returns score and decision only.
   - `Deduplicate(biometricCapture) → DeduplicationResult` — 1:N search; returns hit/no-hit and subject reference.
   - `Revoke(templateId)` — deletes template on identity revocation.

2. **The Identity Core stores only a `templateId` (opaque reference)**, never a biometric template or raw capture.

3. **Network isolation**: the Biometric Vault is on a dedicated network segment. Only the Identity Core service account has inbound access. No direct access from the API layer or any other service.

4. **Encryption**: templates are encrypted at rest using AES-256 with keys managed by a dedicated HSM. The vault holds no key material — keys are fetched from the KMS at startup and held in memory only.

5. **Audit**: every vault operation (enroll, verify, deduplicate, revoke) is logged with actor, timestamp, subject reference and outcome. Logs are shipped to the central SIEM in real time.

6. **Algorithm agnosticism**: the vault wraps biometric SDK calls behind an internal interface (`IBiometricEngine`). The underlying algorithm provider can be replaced without changing the vault API or the Identity Core.

### Deployment diagram

```
┌─────────────────────────────────────┐
│  Application Zone                   │
│  ┌─────────────┐                    │
│  │ Identity    │──── templateId ───▶│
│  │ Core        │◀─── MatchResult ───│
│  └──────┬──────┘                    │
│         │ mTLS only                 │
└─────────┼─────────────────────────┘
          │
┌─────────┼─────────────────────────┐
│  Biometric Vault Zone (isolated)  │
│  ┌──────▼──────┐  ┌─────────────┐ │
│  │ Vault API   │──│ Biometric   │ │
│  │             │  │ SDK         │ │
│  └──────┬──────┘  └─────────────┘ │
│         │                          │
│  ┌──────▼──────┐  ┌─────────────┐ │
│  │ Encrypted   │  │ KMS (HSM)   │ │
│  │ Template DB │  │             │ │
│  └─────────────┘  └─────────────┘ │
└───────────────────────────────────┘
```

## Consequences

- **Blast radius reduction**: a breach of the main identity database does not expose biometric data.
- **Regulatory compliance**: biometric data is physically and logically separated, simplifying GDPR Article 9 compliance documentation.
- **Operational complexity**: two separate systems to deploy, monitor and backup. Vault availability becomes a dependency for enrollment and verification flows.
- **Latency**: biometric verification adds a network hop; p95 latency budget must account for this. Synchronous flows are acceptable for verification; asynchronous processing is preferred for deduplication.
- **SDK upgrades**: replacing the biometric algorithm requires a re-enrollment campaign (existing templates must be re-generated). This is a planned operational event, not an emergency.
