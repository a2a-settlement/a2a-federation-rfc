# A2A Improvement Proposal (AIP) Governance

**Version:** 0.1.0  
**Status:** Draft  
**Effective:** Upon ratification by founding consortium

## 1. Overview

The A2A Settlement Federation Protocol evolves through A2A Improvement Proposals (AIPs). This document defines the process for submitting, reviewing, and ratifying changes to the protocol specification.

The governance model follows the EIP/BIP pattern (Ethereum/Bitcoin Improvement Proposals) weighted by economic activity to resist Sybil attacks.

## 2. Proposal Types

### 2.1 Core Protocol Changes

Changes to fundamental protocol elements:

- Trust Discount interface (inputs, outputs, negotiation)
- Attestation schema (VC types, credential structure)
- Peering handshake protocol
- Identity model (DID methods, key rotation)
- Federation health contract
- Capability manifest schema

**Requirements:**
- Rough consensus from active federated nodes, weighted by 90-day cross-exchange settlement volume
- Breaking changes require supermajority (>66% by volume weight)
- 90-day review period from submission
- Mandatory implementation on at least one testnet before ratification
- Breaking changes include a 180-day deprecation window for the prior version

### 2.2 Extension Proposals

Optional additions that do not modify core protocol behavior:

- New capability schemas for specialized AI task types
- Specialized Trust Discount algorithms
- New evidence artifact formats
- Domain-specific attestation types

**Requirements:**
- Permissionless submission — any developer may propose
- No network-wide consensus required
- Peers selectively opt in by adding the extension to their capability manifest
- Extensions achieving >50% of active volume may be promoted to core via the AIP process

## 3. Proposal Lifecycle

```
Draft → Review → Last Call → Accepted → Final
                    ↓
                 Rejected
```

### 3.1 Draft

Author submits a pull request to this repository containing:

| File | Required | Contents |
|------|----------|----------|
| `aips/AIP-NNNN.md` | Yes | Specification, rationale, backwards-compatibility analysis |
| `schemas/*.json` | If applicable | JSON Schema additions or modifications |
| `test-vectors/*` | If applicable | Reference inputs/outputs for validation |
| `algorithms/*.md` | If applicable | Algorithm specification for Trust Discount registry |

The AIP number is assigned upon PR creation. The PR description must include:

- **Abstract:** One-paragraph summary
- **Motivation:** Why this change is needed
- **Specification:** Precise technical changes
- **Rationale:** Design decisions and alternatives considered
- **Backwards Compatibility:** Impact on existing implementations
- **Test Cases:** How compliance can be verified
- **Reference Implementation:** Link to at least one implementation (may be WIP)

### 3.2 Review

- Review period: **90 days** from the PR being marked as "Review"
- Consortium members and community provide feedback via PR comments
- Author iterates on the proposal based on feedback
- At least one independent testnet implementation must be demonstrated before advancing

### 3.3 Last Call

- **14-day** final comment period
- No substantive changes allowed during Last Call
- Volume-weighted vote tallied from active federated nodes

### 3.4 Accepted / Rejected

- **Core proposals:** Accepted with rough consensus by volume weight (>50% for non-breaking, >66% for breaking)
- **Extensions:** Accepted upon merge (no volume vote required)
- Rejected proposals may be resubmitted after addressing feedback

### 3.5 Final

- Merged into the specification
- Assigned a version number following semantic versioning
- Implementations given the deprecation window (if breaking) to update

## 4. Volume-Weighted Consensus

### 4.1 Rationale

A purely democratic one-node-one-vote system is vulnerable to Sybil attacks. An adversary could spin up many empty exchanges to hijack the protocol. Weighting by economic activity ensures that nodes with real settlement volume — and therefore real economic incentive to maintain protocol security — have proportional influence.

### 4.2 Calculation

```
vote_weight(exchange) = exchange.cross_exchange_volume_ate_90d / sum(all_exchanges.cross_exchange_volume_ate_90d)
```

### 4.3 Minimum Participation

A vote is valid only if participating exchanges represent at least **33%** of total 90-day cross-exchange settlement volume.

## 5. Founding Consortium Provisions

### 5.1 Founding Members

| Member | Entity |
|--------|--------|
| A2A-SE | Truthsetter LLC |
| AgentUnion | bappybot |
| AgentTrustCompany | ATC-Concierge |

### 5.2 Veto Authority

During the permissioned consortium phase, each founding member holds **veto authority** over core protocol changes. A single founding member veto blocks a proposal regardless of volume-weighted consensus.

### 5.3 Veto Sunset

The founding consortium veto **automatically sunsets** when the federation reaches the maturity threshold:

- **10 or more** active federated peers, AND
- **100,000 ATE** or more in cumulative cross-exchange settlement volume

After sunset, governance transitions to pure volume-weighted consensus as defined in Section 4.

### 5.4 Emergency Provisions

Any founding member may invoke an **emergency freeze** on a ratified AIP for up to 30 days if a critical security vulnerability is discovered post-ratification. The freeze must be accompanied by a public disclosure and remediation plan.

## 6. Algorithm Registry Governance

The Trust Discount algorithm registry follows a distinct process:

1. Author submits algorithm specification as a PR to `algorithms/`
2. Specification must include: mathematical definition, input/output contract compliance proof, reference implementation, and test vectors
3. Consortium members review for correctness and interoperability
4. Upon approval, a URN is assigned: `urn:a2a:trust:discount:<algorithm-id>`
5. The registry is **append-only** — deprecated algorithms are marked but never removed

## 7. Versioning

The specification follows [Semantic Versioning 2.0.0](https://semver.org/):

- **MAJOR:** Breaking changes to core protocol (new peering handshake version, incompatible VC schema changes)
- **MINOR:** Backwards-compatible additions (new optional fields, new extension types)
- **PATCH:** Clarifications, typo fixes, test vector additions

## 8. Intellectual Property

All contributions to this repository are licensed under the [Apache License 2.0](LICENSE). Contributors retain copyright but grant the license to all users. No patent encumbrance is permitted on core protocol elements.
