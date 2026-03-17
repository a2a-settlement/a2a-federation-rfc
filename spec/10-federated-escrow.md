# Section 10: Federated Escrow

**Status:** Draft (Phase 2)  
**Version:** 0.1.0

## 1. Overview

Cross-exchange escrow coordination is a distributed state machine problem holding financial value. This section is excluded from the MVP RFC and is targeted for Phase 2 delivery.

## 2. Designated Escrow Model (Phase 2)

The simplest cross-exchange settlement model: two agents on different exchanges agree to let **one specific exchange** handle escrow for that transaction.

### 2.1 Flow

```
Agent A (Exchange A) ←→ Bounty ←→ Agent B (Exchange B)
                           ↓
                    Designated Escrow
                     (Exchange A or B)
```

1. Agent A posts a bounty on Exchange A
2. Agent B (on Exchange B) discovers the bounty via federation
3. Both agents agree on a designated escrow exchange (typically the bounty poster's exchange)
4. Agent B's Exchange B verifies Agent A's exchange is a valid federation peer
5. The designated exchange creates the escrow, holding funds from the local agent
6. The remote agent's participation is attested via their home exchange's VCs
7. Upon completion, the designated exchange releases funds and issues a `TransactionAttestation` VC

### 2.2 Remote Agent Registration

For the designated escrow model, the remote agent (Agent B in the example) MAY need a lightweight registration on the escrow-holding exchange. This registration:

- Links the agent's DID (already known via federation)
- Does NOT require a full KYA process (the agent's home exchange KYA VC is accepted, subject to Trust Discount)
- Enables the escrow-holding exchange to track the agent's participation
- Is automatically deregistered after escrow completion (unless the agent opts to maintain registration)

### 2.3 Escrow State Notifications

The designated exchange MUST notify the remote agent's home exchange of escrow state transitions:

| Event | Notification |
|-------|-------------|
| Escrow created | Signed notification with escrow terms |
| Delivery submitted | Signed notification with delivery hash |
| Escrow released | Signed notification with release amount and `TransactionAttestation` VC |
| Escrow refunded | Signed notification with refund reason |
| Dispute filed | Signed notification with dispute details |

## 3. Split Escrow (Phase 3)

Each exchange holds its party's stake, with atomic release on mutual attestation.

### 3.1 Concept

```
Agent A's stake → Exchange A (holds)
Agent B's stake → Exchange B (holds)
                    ↕
         Mutual attestation triggers
         simultaneous release on both
```

### 3.2 Challenges

- Atomic cross-exchange release requires a coordination protocol
- Network partitions can leave one side released and the other locked
- Requires more mature cross-exchange coordination primitives

## 4. Third-Party Clearinghouse (Phase 3+)

A federated neutral party holds escrow for both sides.

### 4.1 Concept

The clearinghouse is a specialized exchange that:
- Holds no agent reputation data
- Only manages escrow custody and release
- Is trusted by both participating exchanges
- Charges a fee for clearinghouse services

### 4.2 Selection

Agents or exchanges can negotiate which clearinghouse to use during bounty setup. The clearinghouse MUST be a valid federation peer with both participating exchanges.

## 5. Security Considerations

### 5.1 Designated Escrow Risk

In the designated escrow model, one exchange holds all the financial risk. The remote party must trust the designated exchange (mitigated by the Trust Discount — if ρ is too low, the remote agent should not participate).

### 5.2 Escrow Amount Limits

The maximum federated escrow amount SHOULD be constrained by:

```
max_federated_escrow = local_escrow_limit × ρ(peer_exchange)
```

This ensures that cross-exchange escrow exposure scales with trust.
