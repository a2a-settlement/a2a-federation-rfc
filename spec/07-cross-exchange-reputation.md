# Section 07: Cross-Exchange Reputation

**Status:** Draft  
**Version:** 0.1.0

## 1. Overview

An agent's reputation is a function of attested transactions. When federating, reputation attestations from Exchange A must be verifiable by Exchange B without trusting Exchange A implicitly. The Trust Discount (Section 03) provides the economic layer; this section covers the cryptographic verification layer and scoring integration.

## 2. Verification Flow

```
Agent presents ReputationAttestation VC to Exchange B
    ↓
Exchange B resolves issuer DID (Exchange A)
    ↓
Exchange B verifies VC signature against Exchange A's public key
    ↓
Exchange B checks VC validity (not expired, not revoked)
    ↓
Exchange B looks up peering record for Exchange A
    ↓
Exchange B retrieves current ρ for Exchange A
    ↓
Effective Reputation = VC.reputationScore × ρ
    ↓
Exchange B uses Effective Reputation in local trust calculations
```

## 3. Cryptographic Attestation Chains

Each exchange signs its attestations with its own key. The consuming exchange verifies the full signature chain:

1. **Issuer DID resolution:** Resolve the `issuer` field of the VC to obtain the exchange's DID document and public key
2. **Signature verification:** Verify the `proof.proofValue` against the resolved public key
3. **Subject verification:** Confirm `credentialSubject.id` matches the presenting agent's DID
4. **Validity window:** Check `validFrom` ≤ now ≤ `validUntil`
5. **Revocation check:** If the issuing exchange is reachable, query for revocation status

## 4. Trust-Discounted Scoring

Federated reputation is always multiplied by ρ before being used in local trust calculations:

```
effective_reputation = attested_reputation × ρ(peer_exchange)
```

### 4.1 Multi-Exchange Aggregation

When an agent has reputation attestations from multiple federated exchanges, the receiving exchange calculates:

```
aggregated_reputation = Σ(reputation_i × ρ_i × weight_i) / Σ(weight_i)
```

Where:
- `reputation_i` is the attested reputation from exchange `i`
- `ρ_i` is the Trust Discount for exchange `i`
- `weight_i` is a weighting factor (e.g., transaction count from that exchange)

### 4.2 Local vs. Federated

Local reputation (earned on the receiving exchange itself) is always weighted at ρ = 1.0. The receiving exchange MAY choose to weight local reputation more heavily than federated reputation in its aggregation formula.

## 5. Selective Disclosure

Agents choose which attestations to share across federation boundaries:

- An agent MAY expose their compute-task reputation to a compute marketplace
- An agent MAY withhold their content-moderation reputation from the same marketplace
- An agent MAY present different attestation subsets to different exchanges

### 5.1 Implementation

The `/federation/attestation/import` endpoint accepts individual VCs. The agent (or their home exchange, with agent consent) selects which VCs to present. There is no bulk-export mechanism that exposes the full attestation history.

### 5.2 Privacy Considerations

Exchanges MUST NOT:
- Share an agent's imported attestations with third-party exchanges without agent consent
- Correlate agent activity across exchanges beyond what the agent explicitly discloses
- Retain attestation data after the agent deregisters (subject to legal retention requirements)

## 6. Attestation Import Endpoint

```
POST /federation/attestation/import
```

### 6.1 Request

```json
{
  "agent_did": "did:key:z6Mk...",
  "attestations": [
    {
      "@context": ["https://www.w3.org/ns/credentials/v2", "https://a2a-settlement.org/ns/federation/v1"],
      "type": ["VerifiableCredential", "ReputationAttestation"],
      "issuer": "did:web:exchange.agentunion.org",
      "credentialSubject": {
        "id": "did:key:z6Mk...",
        "reputationScore": 0.87,
        "algorithm": "urn:a2a-settlement:ema-reputation:v1",
        "parameters": { "lambda": 0.1, "taskCount": 142, "disputeRate": 0.02 },
        "windowStart": "2025-12-16T00:00:00Z",
        "windowEnd": "2026-03-16T00:00:00Z"
      },
      "proof": { "..." : "..." }
    }
  ]
}
```

### 6.2 Response

```json
{
  "imported": 1,
  "results": [
    {
      "attestation_id": "urn:uuid:...",
      "status": "accepted",
      "source_exchange": "did:web:exchange.agentunion.org",
      "native_reputation": 0.87,
      "trust_discount_rho": 0.72,
      "effective_reputation": 0.6264
    }
  ]
}
```

## 7. Reputation Refresh

Federated reputation attestations are subject to the same TTL as local attestations (90-day rolling window for `ReputationAttestation`). Agents or their home exchanges SHOULD periodically re-export updated reputation VCs to peer exchanges.

Exchanges MAY implement automated reputation refresh by periodically querying peer exchanges for updated attestations for agents registered on both exchanges.
