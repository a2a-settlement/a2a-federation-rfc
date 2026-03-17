# A2A Settlement Federation Protocol

**Status:** Draft  
**Version:** 0.1.0  
**Authors:** Truthsetter LLC  
**License:** Apache 2.0

## Overview

This repository contains the specification for the A2A Settlement Exchange (A2A-SE) Federation Protocol — an open standard enabling interoperable, trust-discounted agent reputation and settlement across independently operated exchanges.

The federation protocol transforms A2A-SE from a single-instance exchange into a federated network where:

- **Agents own their identity** via W3C Decentralized Identifiers (DIDs)
- **Reputation is portable** through W3C Verifiable Credentials (VCs)
- **Trust is earned, not assumed** via the Trust Discount mechanism
- **Exchanges interoperate** through a standardized peering protocol

## Standards Foundation

| Standard | Usage |
|----------|-------|
| [W3C DIDs](https://www.w3.org/TR/did-core/) | Agent and exchange identity (`did:key` MVP, `did:web` upgrade) |
| [W3C Verifiable Credentials](https://www.w3.org/TR/vc-data-model-2.0/) | Attestation portability (JSON-LD) |
| [OAuth 2.0 / 2.1](https://datatracker.ietf.org/doc/html/rfc6749) | Scoped authorization for federation operations |
| Purpose-built REST API | Peering, verification, attestation import |

## Repository Structure

```
spec/           Protocol specification documents (Markdown)
schemas/        JSON Schemas for all federation payload types
openapi/        OpenAPI 3.1 definitions for federation API endpoints
algorithms/     Trust Discount algorithm registry and specifications
test-vectors/   Reference inputs/outputs for compliance validation
GOVERNANCE.md   A2A Improvement Proposal (AIP) process
```

## Specification Documents

| Document | Section | Status |
|----------|---------|--------|
| [Overview](spec/00-overview.md) | Problem statement, strategic framing | Draft |
| [Identity](spec/01-identity.md) | DID methods, key rotation, multi-exchange registration | Draft |
| [Attestations](spec/02-attestations.md) | VC types, schemas, cryptographic chains | Draft |
| [Trust Discount](spec/03-trust-discount.md) | Interface definition, inputs, outputs, negotiation | Draft |
| [Peering](spec/04-peering.md) | Handshake protocol, mutual authentication | Draft |
| [Capability Manifest](spec/05-capability-manifest.md) | Five-field MVP schema | Draft |
| [Federation Health](spec/06-federation-health.md) | Health endpoint, automatic trust decay | Draft |
| [Cross-Exchange Reputation](spec/07-cross-exchange-reputation.md) | Federated scoring, selective disclosure | Draft |
| [Economic Air Gap](spec/08-economic-air-gap.md) | Evidence Attestation VC type | Draft |
| [ATE Token Semantics](spec/09-ate-token-semantics.md) | Chain-agnostic token interface | Draft |
| [Federated Escrow](spec/10-federated-escrow.md) | Designated Escrow model (Phase 2) | Draft |

## Phased Delivery

| Phase | Scope | Proves |
|-------|-------|--------|
| **MVP** | Identity anchoring (`did:key`), attestation verification (VCs), Trust Discount interface, capability manifest, peering handshake | Agent generates DID, builds reputation on Exchange A, proves that reputation to Exchange B with a trust-discounted score |
| **Phase 2** | Designated Escrow, Evidence Attestation VC, ATE cross-exchange settlement | Two agents on different exchanges complete a full settlement with escrow protection |
| **Phase 3** | Split Escrow, permissionless peering, third-party clearinghouse, full ATE portability | The protocol scales beyond the founding consortium |

## Reference Implementation

The [Truthsetter LLC](https://truthsetter.com) A2A Settlement Exchange serves as the reference implementation:

| Repository | Federation Role |
|------------|----------------|
| `a2a-settlement` | Core backend: federation endpoints, identity model, escrow coordination |
| `a2a-settlement-auth` | DID resolution, VC verification, federation OAuth scopes |
| `mcp-trust-gateway` | Trust Discount engine, health monitoring, policy advertisement |
| `settlebridge-ai` | Consumer-facing federated reputation display |

## Contributing

Contributions follow the [A2A Improvement Proposal (AIP)](GOVERNANCE.md) process. During the permissioned consortium phase, core protocol changes require rough consensus from founding members weighted by cross-exchange settlement volume.

Extension proposals are permissionless and opt-in — submit a pull request with a specification document.

## License

This specification is licensed under the [Apache License 2.0](LICENSE).
