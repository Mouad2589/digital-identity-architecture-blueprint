# Roadmap

## Phase 1 — Foundation (complete)

- [x] Repository structure and governance files.
- [x] Initial README with architecture overview and Mermaid diagram.
- [x] ADR-0001: public repository scope.
- [x] Context document: problem domain, stakeholders, quality attributes.
- [x] Modules document: all major modules described with design concerns.
- [x] Security model: principles, IAM, data protection, network model, threat model.
- [x] Interoperability document: protocols, gateway architecture, API design principles.
- [x] Operational view: deployment, observability, incident management, backup/recovery.

## Phase 2 — Depth (in progress)

- [x] ADR-0002: identity assurance level strategy.
- [x] ADR-0003: biometric data isolation architecture.
- [x] ADR-0004: eCVRS civil registration integration.
- [x] ADR-0005: offline-first architecture for low-connectivity environments.
- [x] ADR-0006: CRDT sync conflict resolution.
- [x] eCVRS module: full architecture, flows, data model, security model.
- [x] Offline-first design: connectivity tiers, sync engine, SMS fallback, physical transport, offline verification, power resilience, device PKI, chaos testing.
- [x] Detailed sequence diagram: enrollment flow (operator-assisted).
- [x] Detailed sequence diagram: birth registration (offline field flow).
- [x] Detailed sequence diagram: birth registration (connected district flow).
- [x] Detailed sequence diagram: death registration and identity revocation chain.
- [x] Detailed sequence diagram: offline certificate verification.
- [ ] ADR-0007: cross-border federation approach (eIDAS 2.0).
- [ ] OpenAPI specification for the identity verification endpoint.
- [ ] Detailed C4 level-3 diagram for Identity Core.
- [ ] Detailed sequence diagram: citizen authentication flow (OIDC + step-up).
- [ ] Detailed sequence diagram: document issuance flow.
- [ ] STRIDE threat model for the enrollment flow.
- [ ] STRIDE threat model for the offline sync flow.

## Phase 3 — Public authority

- [ ] Link repository to a LinkedIn article or case study.
- [ ] Add a blog-post style narrative: "How I approach digital identity architecture for African deployments".
- [ ] Publish SLO dashboard design (Grafana screenshot or template).
- [ ] Add a comparison: centralised vs federated vs self-sovereign identity models.
- [ ] Add a section: offline-first vs online-first trade-off analysis.
- [ ] Release notes for each major documentation improvement.
