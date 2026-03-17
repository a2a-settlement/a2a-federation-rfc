# Section 05: Capability Manifest

**Status:** Draft  
**Version:** 0.1.0

## 1. Overview

The capability manifest is a machine-readable declaration of an exchange's federation capabilities. It enables automated compatibility checking between peers, allowing the transition to permissionless peering to become a matter of programmatic verification rather than human negotiation.

## 2. MVP Schema

The MVP manifest requires exactly five fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `node_did` | string (DID) | Yes | The exchange's own decentralized identifier |
| `supported_assets` | string[] | Yes | Settlement tokens the exchange can custody |
| `attestation_types` | string[] | Yes | VC types the exchange can issue and verify |
| `policy_uri` | URI | Yes | URL for the exchange's Trust Discount policy |
| `endpoints` | object | Yes | Base URLs for federation API endpoints |

## 3. Schema Definition

```json
{
  "node_did": "did:web:exchange.a2a-settlement.org",
  "supported_assets": [
    "a2a:ledger:ate",
    "eip155:8453/erc20/0xATE_CONTRACT_ADDRESS"
  ],
  "attestation_types": [
    "IdentityAttestation",
    "CapabilityAttestation",
    "ReputationAttestation",
    "EvidenceAttestation",
    "TransactionAttestation"
  ],
  "policy_uri": "https://exchange.a2a-settlement.org/.well-known/a2a-trust-policy.json",
  "endpoints": {
    "verify": "https://exchange.a2a-settlement.org/federation/verify",
    "peer": "https://exchange.a2a-settlement.org/federation/peer",
    "health": "https://exchange.a2a-settlement.org/.well-known/a2a-federation-health",
    "attestation_import": "https://exchange.a2a-settlement.org/federation/attestation/import"
  }
}
```

## 4. Asset Identifiers

### 4.1 Ledger-Based (Non-Chain)

```
a2a:ledger:ate
```

For exchanges using traditional ledger-based ATE accounting without a blockchain settlement layer.

### 4.2 EVM-Based

Following [CAIP-19](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-19.md):

```
eip155:<chain_id>/erc20/<contract_address>
```

Example: `eip155:8453/erc20/0x1234...` (ATE on Base)

### 4.3 Solana SPL

```
solana:<cluster>/spl/<mint_address>
```

## 5. Endpoint Requirements

All federated exchanges MUST implement:

| Endpoint | Path | Method | Purpose |
|----------|------|--------|---------|
| `verify` | `/federation/verify` | POST | Verify a VC presented by a federated agent |
| `peer` | `/federation/peer` | POST | Peering handshake |
| `health` | `/.well-known/a2a-federation-health` | GET | Health telemetry |
| `attestation_import` | `/federation/attestation/import` | POST | Import cross-exchange attestation |

## 6. Manifest Exchange

The manifest is exchanged during the `/federation/peer` handshake and cached by the receiving exchange. Changes to the manifest trigger a re-peering notification so that capability routing stays current.

### 6.1 Compatibility Checking

When an agent on Exchange A attempts to interact with Exchange B, the compatibility check verifies:

1. Exchange B's manifest includes the required `attestation_types` for the operation
2. Exchange B supports a compatible `supported_assets` entry for settlement
3. Exchange B's `policy_uri` is reachable and contains a parseable Trust Discount policy
4. All required `endpoints` are reachable

### 6.2 Caching

Exchanges SHOULD cache peer manifests with a TTL of 1 hour. Manifest changes trigger re-peering, which invalidates the cache.
