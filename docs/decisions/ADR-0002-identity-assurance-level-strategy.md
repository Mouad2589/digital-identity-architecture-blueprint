# ADR-0002 — Identity Assurance Level Strategy

## Status

Accepted

## Context

A digital identity platform serves operations with vastly different risk profiles: reading public information requires minimal assurance, while signing a legal document or accessing medical records requires high assurance. Using a single assurance level for all operations either over-secures low-risk flows (degrading UX) or under-secures high-risk flows (creating unacceptable exposure).

The platform must align with internationally recognised frameworks:

- **NIST SP 800-63-3** defines three Identity Assurance Levels (IAL1, IAL2, IAL3) and Authentication Assurance Levels (AAL1, AAL2, AAL3).
- **eIDAS 2.0** defines Low, Substantial, and High assurance levels.

## Decision

The platform adopts a **three-tier assurance model** mapped to both NIST 800-63 and eIDAS:

| Internal Level | eIDAS | NIST IAL | NIST AAL | Proofing required | Authentication |
|---|---|---|---|---|---|
| LoA 1 — Low | Low | IAL1 | AAL1 | Self-asserted, email verified | Single factor (password) |
| LoA 2 — Substantial | Substantial | IAL2 | AAL2 | Remote identity proofing + document check | MFA (TOTP or push) |
| LoA 3 — High | High | IAL3 | AAL3 | In-person proofing + biometric verification | Hardware key (FIDO2 / smart card) |

### Assurance level assignment per operation type

| Operation | Required LoA |
|---|---|
| View public information | LoA 1 |
| Access citizen portal (read) | LoA 2 |
| Submit enrollment request | LoA 2 |
| Request identity document | LoA 2 |
| Sign legal document | LoA 3 |
| Access medical or tax records | LoA 3 |
| Admin operations | LoA 3 |

### Enforcement mechanism

- Assurance level is encoded as a claim (`acr`) in the JWT issued by the OIDC provider.
- Each API endpoint declares its minimum required `acr` value.
- The authorization middleware rejects tokens with insufficient `acr` before the request reaches the application layer.
- Step-up authentication is triggered when a citizen tries to access a higher-assurance resource without meeting the current requirement.

## Consequences

- Citizens enrolled at LoA 2 can access most services without re-enrolling.
- A separate high-assurance enrollment flow is required for LoA 3 operations.
- API consumers must declare their required assurance level when registering as a relying party.
- The step-up flow adds friction for citizens but is a deliberate security trade-off for high-risk operations.
- Audit events must record the assurance level at which each sensitive operation was performed.
