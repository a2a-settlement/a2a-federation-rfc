# Section 09: ATE Token Semantics

**Status:** Draft  
**Version:** 0.1.0

## 1. Overview

If ATE tokens are locked to a single exchange, the federation has disconnected company towns, not an economy. However, mandating a specific blockchain in the RFC would create a hard infrastructure dependency that slows adoption. This section defines chain-agnostic ATE token semantics.

## 2. Two-Layer Approach

### 2.1 RFC Layer (Chain-Agnostic)

The specification defines a standard interface for ATE operations. The RFC is opinionated about **semantics** (what ATE represents and how it flows) but agnostic about **substrate** (where and how ATE is stored).

### 2.2 Reference Implementation Layer

Reference implementations choose specific settlement layers:

| Implementation | Substrate |
|----------------|-----------|
| Ledger-based | Traditional database ledger with cryptographic receipts |
| EVM-based | ERC-20 on Base, Arbitrum, or other L2 |
| Solana-based | SPL token |

## 3. Standard ATE Interface

Every federated exchange MUST implement the following logical operations:

### 3.1 Balance

Query an agent's ATE balance.

```
balance(agent_did) → { available: float, held_in_escrow: float, total: float }
```

### 3.2 Transfer

Transfer ATE between agents on the same exchange.

```
transfer(from_did, to_did, amount, reference) → { transaction_id: string, new_balance: float }
```

### 3.3 Stake

Lock ATE as a stake (for reputation bonding, exchange operator deposits, etc.).

```
stake(agent_did, amount, purpose, duration) → { stake_id: string, locked_until: datetime }
```

### 3.4 Escrow Lock

Lock ATE into an escrow for a specific settlement.

```
escrow_lock(payer_did, amount, escrow_terms) → { escrow_id: string, locked_amount: float }
```

### 3.5 Escrow Release

Release ATE from escrow to the designated recipient.

```
escrow_release(escrow_id, recipient_did, amount) → { transaction_id: string }
```

## 4. Cross-Exchange Settlement

### 4.1 Token Portability

An agent can stake ATE on Exchange A and use those same tokens to settle a bounty on Exchange B. The mechanism depends on the substrate:

- **Ledger-based:** Inter-exchange settlement via bilateral netting (periodic reconciliation between exchanges)
- **Chain-based:** Direct on-chain transfer between exchange custody addresses

### 4.2 Asset Identification

Cross-exchange references use [CAIP-19](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-19.md) identifiers in the capability manifest's `supported_assets` field:

```json
{
  "supported_assets": [
    "a2a:ledger:ate",
    "eip155:8453/erc20/0xATE_ADDRESS"
  ]
}
```

### 4.3 Compatibility

Two exchanges can settle cross-exchange transactions only if their `supported_assets` sets overlap. The capability manifest enables automated pre-flight compatibility checking.

## 5. Escrow in Federated Context

See [Section 10: Federated Escrow](10-federated-escrow.md) for the Designated Escrow model and cross-exchange escrow coordination.
