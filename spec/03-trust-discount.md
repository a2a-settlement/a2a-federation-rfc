# Section 03: Trust Discount Interface

**Status:** Draft  
**Version:** 0.1.0

## 1. Overview

Verifiable Credentials prove **authenticity** (the attestation was not tampered with), but they do not prove **reliability** (the issuing exchange has good judgment). The Trust Discount bridges this gap as a middleware function evaluated by the receiving exchange whenever it imports a federated reputation score.

The RFC defines the **interface**, not the implementation. This follows the TLS cipher suite model: define the contract so that different exchanges can implement their own discount algorithms while remaining fully interoperable.

## 2. Standard Inputs (Mandatory Telemetry)

All federated A2A-SE nodes MUST expose the following telemetry metrics to their peers. These form the mandatory inputs for any valid Trust Discount algorithm:

| Field | Type | Description |
|-------|------|-------------|
| `federation_age_days` | integer | Days elapsed since the initial cryptographic peering handshake between Exchange A and Exchange B |
| `cross_exchange_volume_ate` | float | Total financial volume (ATE-denominated) of successfully settled, undisputed transactions between agents of the two exchanges |
| `cross_exchange_tx_count` | integer | Raw count of successful cross-boundary settlements |
| `attestation_success_rate` | float | Ratio of valid, undisputed attestations vs. revoked/disputed attestations from the peer exchange over a 90-day rolling window |

### 2.1 Recommended Extension Inputs

These inputs are OPTIONAL but RECOMMENDED:

| Field | Type | Description |
|-------|------|-------------|
| `peer_stake_slashing_events` | integer | Number of times the peer exchange has caught and slashed malicious actors. Counter-intuitively, occasional slashing proves the immune system works. |
| `uptime_90d` | float | 90-day rolling uptime percentage from the federation health endpoint |
| `avg_attestation_latency_ms` | integer | Average verification engine speed from the federation health endpoint |

## 3. Standard Output

The RFC strictly mandates the output format:

### 3.1 The Multiplier

The algorithm outputs a floating-point multiplier, **ρ** (rho), strictly bounded:

```
0.0 ≤ ρ ≤ 1.0
```

### 3.2 Application Formula

```
Effective Reputation = Native Attested Reputation × ρ
```

### 3.3 State Semantics

| ρ Value | Meaning |
|---------|---------|
| `1.0` | Full parity — peer exchange trusted exactly as much as local exchange |
| `0.1–0.9` | Discounted trust — agent capabilities/escrow limits are constrained |
| `0.0` | Quarantine / null route — attestations carry zero economic weight |

## 4. Protocol Negotiation

Exchanges negotiate Trust Discount parameters during the `/federation/peer` handshake. Each exchange advertises its policy transparently:

```json
{
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
  }
}
```

### 4.1 Transparency Requirements

- The policy MUST be machine-readable
- The policy MUST be published at `/.well-known/a2a-trust-policy.json`
- Agents on Exchange A MUST be able to determine how their reputation translates on Exchange B **before** attempting any cross-exchange activity
- Policy changes MUST trigger re-peering notification to affected peers

### 4.2 Pre-Flight Calculation

Given a published Trust Discount policy, any agent or exchange can compute:

```
effective_reputation = agent_native_reputation × rho(peer_telemetry)
```

UX surfaces SHOULD display: "Your effective reputation on this exchange would be 72/100 (0.85× discount applied)."

## 5. Sybil Resistance

The Trust Discount provides economic Sybil resistance without requiring proof-of-work or proof-of-stake:

1. A malicious actor spins up a new exchange and issues perfect attestations
2. Every federated peer applies a near-zero ρ because:
   - `federation_age_days` ≈ 0
   - `cross_exchange_volume_ate` = 0
   - `cross_exchange_tx_count` = 0
3. The attacker cannot fast-forward time or fabricate real settlement volume
4. Trust must be earned through genuine economic activity over time

## 6. Algorithm Registry

See [algorithms/](../algorithms/) for the full registry.

### 6.1 Registry Model

**Centralized vocabulary, decentralized execution.** This mirrors the IANA registry model for TLS cipher suites.

- The RFC maintains an append-only registry of approved algorithm URNs
- Each exchange chooses and publishes its algorithm via `/.well-known/a2a-trust-policy.json`
- New algorithms are submitted via pull request with specification, test vectors, and reference implementation

### 6.2 URN Format

```
urn:a2a:trust:discount:<algorithm-id>
```

Examples:
- `urn:a2a:trust:discount:linear-volume-weighted-v1`
- `urn:a2a:trust:discount:step-function-v1`
- `urn:a2a:trust:discount:exponential-decay-v1`

## 7. Integration with Escrow

The Trust Discount constrains an agent's effective capabilities on a remote exchange:

- **Escrow limits:** Maximum escrow amount = local limit × ρ
- **Bidding eligibility:** Bounties with `min_reputation > effective_reputation` are unavailable
- **Capability gates:** Higher-trust operations require higher ρ thresholds

## 8. Algorithm Compliance

A Trust Discount algorithm implementation is compliant if and only if:

1. It accepts the four mandatory input fields (Section 2)
2. It outputs a single float ρ in [0.0, 1.0] (Section 3)
3. It is monotonically non-decreasing with respect to `cross_exchange_volume_ate` (more volume → higher or equal trust)
4. It produces ρ = 0.0 when all mandatory inputs are zero (new/unknown peers start at zero trust)
5. It passes all test vectors for its registered URN (see `test-vectors/trust-discount/`)
