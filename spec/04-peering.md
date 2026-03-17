# Section 04: Peering Protocol

**Status:** Draft  
**Version:** 0.1.0

## 1. Overview

Peering establishes a cryptographic trust relationship between two federated exchanges. The peering handshake is a mutual authentication process where both exchanges verify each other's identity, exchange capability manifests, and advertise Trust Discount policies.

## 2. Peering Endpoint

```
POST /federation/peer
```

### 2.1 Request

The initiating exchange sends a signed peering request:

```json
{
  "type": "PeeringRequest",
  "initiator": {
    "did": "did:web:exchange.a2a-settlement.org",
    "name": "A2A Settlement Exchange",
    "operator": "Truthsetter LLC"
  },
  "capability_manifest": {
    "node_did": "did:web:exchange.a2a-settlement.org",
    "supported_assets": ["a2a:ledger:ate"],
    "attestation_types": ["IdentityAttestation", "CapabilityAttestation", "ReputationAttestation", "EvidenceAttestation", "TransactionAttestation"],
    "policy_uri": "https://exchange.a2a-settlement.org/.well-known/a2a-trust-policy.json",
    "endpoints": {
      "verify": "https://exchange.a2a-settlement.org/federation/verify",
      "peer": "https://exchange.a2a-settlement.org/federation/peer",
      "health": "https://exchange.a2a-settlement.org/.well-known/a2a-federation-health",
      "attestation_import": "https://exchange.a2a-settlement.org/federation/attestation/import"
    }
  },
  "trust_discount_policy": {
    "algorithm_id": "urn:a2a:trust:discount:linear-volume-weighted-v1",
    "initial_rho": 0.15,
    "parameters": {
      "volume_threshold_ate": 10000,
      "rho_at_threshold": 0.40,
      "max_rho": 0.85,
      "attestation_success_floor": 0.92,
      "review_cadence_days": 30
    }
  },
  "timestamp": "2026-03-16T00:00:00Z",
  "nonce": "random-256-bit-hex",
  "proof": {
    "type": "Ed25519Signature2020",
    "verificationMethod": "did:web:exchange.a2a-settlement.org#key-1",
    "proofValue": "<base58btc-signature>"
  }
}
```

### 2.2 Response

The receiving exchange responds with its own signed manifest:

```json
{
  "type": "PeeringResponse",
  "status": "accepted",
  "responder": {
    "did": "did:web:exchange.agentunion.org",
    "name": "AgentUnion Exchange",
    "operator": "AgentUnion"
  },
  "capability_manifest": { "..." : "..." },
  "trust_discount_policy": { "..." : "..." },
  "peering_id": "urn:uuid:federated-peer-001",
  "effective_from": "2026-03-16T00:00:00Z",
  "timestamp": "2026-03-16T00:01:00Z",
  "nonce": "response-nonce-256-bit-hex",
  "request_nonce": "random-256-bit-hex",
  "proof": {
    "type": "Ed25519Signature2020",
    "verificationMethod": "did:web:exchange.agentunion.org#key-1",
    "proofValue": "<base58btc-signature>"
  }
}
```

### 2.3 Response Status Codes

| Status | Meaning |
|--------|---------|
| `accepted` | Peering established, mutual trust relationship active |
| `pending_review` | Request received, requires manual operator approval |
| `rejected` | Peering denied (reason included in `rejection_reason` field) |

## 3. Mutual Authentication

Both exchanges MUST verify:

1. The peer's DID resolves to a valid DID document
2. The proof signature is valid against the DID's verification method
3. The nonce is fresh (prevents replay attacks)
4. The timestamp is within an acceptable skew window (±5 minutes)

## 4. Peering Lifecycle

```
Initiated → Pending → Active → Suspended → Terminated
                                    ↑           ↑
                                    └── Active ──┘
```

### 4.1 Active

Both exchanges have completed the handshake and are actively exchanging attestations and health telemetry.

### 4.2 Suspended

Triggered by:
- Health check failures (3 consecutive missed checks)
- Manual operator action
- Policy violation detection

During suspension, the Trust Discount ρ decays but the peering relationship is preserved. Resumption restores the relationship with asymmetric ρ recovery.

### 4.3 Terminated

Permanent termination of the peering relationship. Requires a new handshake to re-establish.

## 5. Re-Peering

When a peer's capability manifest changes (new attestation types, endpoint URLs, or Trust Discount policy), the peer MUST send a re-peering notification:

```
POST /federation/peer
```

With the updated manifest. The receiving exchange verifies the update and refreshes its cached peer record. Re-peering does not reset `federation_age_days`.

## 6. Security Considerations

### 6.1 Replay Protection

Each peering request includes a unique nonce and timestamp. The receiving exchange MUST reject requests with:
- Previously seen nonces (within a 24-hour window)
- Timestamps outside the ±5-minute skew window

### 6.2 Man-in-the-Middle

The DID-based mutual authentication prevents MITM attacks: both parties verify the other's signature against their published DID document, which is resolved over TLS.

### 6.3 Exchange Impersonation

An attacker cannot impersonate an exchange without possessing the private key corresponding to the exchange's DID. The `did:web` method's reliance on DNS/TLS provides an additional layer of domain verification.
