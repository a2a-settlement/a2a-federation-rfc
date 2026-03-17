# Section 02: Attestations

**Status:** Draft  
**Version:** 0.1.0

## 1. Overview

The A2A-SE federation maps its attestation chain to standard W3C JSON-LD Verifiable Credentials. Each attestation stage becomes a VC type with a defined schema, and the chain is an ordered set of VCs with cryptographic links between them.

This provides immediate interoperability with any system that can verify W3C VCs.

## 2. VC Types

The federation defines five attestation VC types:

| Type | Description | Issuer | Max TTL | Renewal |
|------|-------------|--------|---------|---------|
| `IdentityAttestation` | KYA identity verification | Exchange | 365 days | Required |
| `CapabilityAttestation` | Agent skill/capability claim | Exchange | 180 days | Required |
| `ReputationAttestation` | EMA reputation score | Exchange | 90 days | Auto-recalculated |
| `EvidenceAttestation` | "Evidence evaluated, found sufficient" | Exchange | Permanent | N/A |
| `TransactionAttestation` | Settlement proof | Exchange | Permanent | N/A |

## 3. Common VC Structure

All federation VCs share a common structure:

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://a2a-settlement.org/ns/federation/v1"
  ],
  "type": ["VerifiableCredential", "<AttestationType>"],
  "id": "urn:uuid:<unique-id>",
  "issuer": "<exchange-did>",
  "validFrom": "<iso8601-datetime>",
  "validUntil": "<iso8601-datetime>",
  "credentialSubject": {
    "id": "<agent-did>",
    ...
  },
  "proof": {
    "type": "Ed25519Signature2020",
    "created": "<iso8601-datetime>",
    "verificationMethod": "<exchange-did>#<key-id>",
    "proofPurpose": "assertionMethod",
    "proofValue": "<base58btc-encoded-signature>"
  }
}
```

### 3.1 Required Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `@context` | array | Yes | Must include W3C VC v2 and A2A-SE federation context |
| `type` | array | Yes | Must include `VerifiableCredential` and the specific type |
| `id` | string | Yes | Globally unique URN (UUID v4 recommended) |
| `issuer` | string | Yes | DID of the issuing exchange |
| `validFrom` | string | Yes | ISO 8601 datetime |
| `validUntil` | string | Conditional | Required for TTL-bound attestations |
| `credentialSubject.id` | string | Yes | DID of the agent |
| `proof` | object | Yes | Ed25519Signature2020 proof |

## 4. Identity Attestation

Issued upon KYA verification. Asserts that the exchange has verified the agent's identity to a specified level.

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://a2a-settlement.org/ns/federation/v1"
  ],
  "type": ["VerifiableCredential", "IdentityAttestation"],
  "id": "urn:uuid:a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "issuer": "did:web:exchange.a2a-settlement.org",
  "validFrom": "2026-03-16T00:00:00Z",
  "validUntil": "2027-03-16T00:00:00Z",
  "credentialSubject": {
    "id": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
    "kyaLevel": 2,
    "verificationMethod": "agent-card-attestation",
    "exchangeAccountId": "agent-xyz-001"
  },
  "proof": { "..." : "..." }
}
```

### 4.1 Credential Subject Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kyaLevel` | integer | Yes | KYA verification level (0–3) |
| `verificationMethod` | string | Yes | How identity was verified |
| `exchangeAccountId` | string | Yes | Agent's account ID on the issuing exchange |

## 5. Capability Attestation

Asserts that the exchange has verified the agent possesses specific capabilities.

```json
{
  "type": ["VerifiableCredential", "CapabilityAttestation"],
  "credentialSubject": {
    "id": "did:key:z6Mk...",
    "capabilities": ["compute", "content-moderation", "code-review"],
    "verifiedAt": "2026-03-16T00:00:00Z",
    "verificationEvidence": "urn:a2a:evidence:capability-test-001"
  }
}
```

## 6. Reputation Attestation

Asserts the agent's EMA reputation score as calculated by the issuing exchange. This is the primary VC type that crosses federation boundaries with Trust Discount applied.

```json
{
  "type": ["VerifiableCredential", "ReputationAttestation"],
  "credentialSubject": {
    "id": "did:key:z6Mk...",
    "reputationScore": 0.87,
    "algorithm": "urn:a2a-settlement:ema-reputation:v1",
    "parameters": {
      "lambda": 0.1,
      "taskCount": 142,
      "disputeRate": 0.02
    },
    "windowStart": "2025-12-16T00:00:00Z",
    "windowEnd": "2026-03-16T00:00:00Z"
  }
}
```

### 6.1 Credential Subject Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reputationScore` | float | Yes | 0.0–1.0 EMA score |
| `algorithm` | string | Yes | URN identifying the scoring algorithm |
| `parameters` | object | Yes | Algorithm parameters for reproducibility |
| `windowStart` | string | Yes | Rolling window start |
| `windowEnd` | string | Yes | Rolling window end |

## 7. Evidence Attestation

Asserts that the exchange evaluated evidence and found it sufficient, **without revealing what the evidence was**. This is the zero-knowledge-like pattern that preserves the Economic Air Gap across federation boundaries.

```json
{
  "type": ["VerifiableCredential", "EvidenceAttestation"],
  "credentialSubject": {
    "id": "did:key:z6Mk...",
    "taskId": "urn:a2a:task:bounty-12345",
    "evaluationOutcome": "sufficient",
    "evaluatorType": "ai-mediator",
    "evidenceHash": "sha256:a1b2c3d4e5f6...",
    "evaluatedAt": "2026-03-16T12:00:00Z"
  }
}
```

The `evidenceHash` allows the agent to prove they possess the original evidence without revealing it. A verifier can request the agent to produce the evidence and verify the hash matches.

### 7.1 Credential Subject Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `taskId` | string | Yes | URN of the task/bounty evaluated |
| `evaluationOutcome` | enum | Yes | `sufficient`, `insufficient`, `partial` |
| `evaluatorType` | string | Yes | `ai-mediator`, `human-arbitrator`, `automated` |
| `evidenceHash` | string | Yes | SHA-256 hash of the evidence bundle |
| `evaluatedAt` | string | Yes | When evaluation occurred |

## 8. Transaction Attestation

Permanent record of a completed settlement.

```json
{
  "type": ["VerifiableCredential", "TransactionAttestation"],
  "credentialSubject": {
    "id": "did:key:z6Mk...",
    "transactionId": "urn:a2a:tx:escrow-67890",
    "role": "provider",
    "amountAte": 500.0,
    "outcome": "completed",
    "counterpartyDid": "did:key:z6Mk...",
    "completedAt": "2026-03-16T15:00:00Z",
    "merkleRoot": "sha256:f1e2d3c4b5a6..."
  }
}
```

## 9. Attestation Chains

Attestations form a cryptographic chain linking identity to reputation to transaction history. The chain is verified by:

1. Resolving the issuer's DID to obtain the public key
2. Verifying the proof signature on each VC
3. Verifying the `credentialSubject.id` matches the agent's DID
4. Checking `validUntil` for TTL-bound attestations
5. Checking revocation status (if the issuing exchange is reachable)

### 9.1 Cross-Reference URIs

Attestations can reference attestations on other exchanges by DID-anchored URI:

```
urn:a2a:attestation:did:web:exchange.a2a-settlement.org:vc:a1b2c3d4
```

This enables independent verification by any party with access to the referenced exchange.

## 10. Revocation

Attestation revocation follows the existing A2A-SE revocation model extended for federation:

1. The issuing exchange publishes a revocation VC
2. Federated peers check revocation status during pull-on-verification
3. If the issuing exchange is unreachable, the cached attestation remains valid until TTL expiry (safe degradation)

Revocation reasons: `key_compromise`, `erroneous_issuance`, `deregistration`, `policy_violation`, `exchange_shutdown`.
