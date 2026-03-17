# Linear Volume-Weighted v1

**URN:** `urn:a2a:trust:discount:linear-volume-weighted-v1`  
**Status:** Active  
**Version:** 1.0.0

## 1. Summary

Rho increases linearly with cross-exchange settlement volume up to a configurable threshold, then remains constant at `max_rho`. A floor on attestation success rate acts as a kill switch — if the peer's attestation quality drops below the floor, rho is forced to zero.

## 2. Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `volume_threshold_ate` | float | Yes | ATE volume at which rho reaches `rho_at_threshold` |
| `rho_at_threshold` | float | Yes | Rho value when volume equals `volume_threshold_ate` |
| `max_rho` | float | Yes | Maximum achievable rho (cap) |
| `attestation_success_floor` | float | Yes | Minimum attestation success rate; below this, rho = 0 |
| `review_cadence_days` | integer | No | Advisory: how often parameters are reviewed |

## 3. Algorithm

```
FUNCTION compute_rho(inputs, params):
    # Kill switch: attestation quality gate
    IF inputs.attestation_success_rate < params.attestation_success_floor:
        RETURN 0.0

    # Base rho from federation age (small time-based component)
    age_factor = MIN(inputs.federation_age_days / 365.0, 1.0) * 0.1

    # Volume-based linear interpolation
    IF inputs.cross_exchange_volume_ate <= 0:
        volume_rho = 0.0
    ELSE IF inputs.cross_exchange_volume_ate >= params.volume_threshold_ate:
        volume_rho = params.rho_at_threshold
    ELSE:
        volume_rho = (inputs.cross_exchange_volume_ate / params.volume_threshold_ate) * params.rho_at_threshold

    # Combined rho with cap
    raw_rho = age_factor + volume_rho
    RETURN MIN(raw_rho, params.max_rho)
```

## 4. Properties

- **Monotonically non-decreasing** with volume: more volume always equals higher or equal trust
- **Zero at zero:** When all inputs are zero, output is 0.0
- **Bounded:** Output is always in [0.0, max_rho] ⊆ [0.0, 1.0]
- **Attestation gate:** Binary kill switch prevents gaming via high volume with bad attestations

## 5. Use Cases

Best suited for exchanges that want:
- Predictable, transparent trust growth
- Easy-to-explain policy for agents ("do X volume, get Y trust")
- Conservative approach with hard quality gate

## 6. Example

Given parameters:
```json
{
  "volume_threshold_ate": 10000,
  "rho_at_threshold": 0.40,
  "max_rho": 0.85,
  "attestation_success_floor": 0.92
}
```

| federation_age_days | cross_exchange_volume_ate | attestation_success_rate | rho |
|--------------------:|-------------------------:|-------------------------:|----:|
| 0 | 0 | 1.00 | 0.000 |
| 30 | 1000 | 0.98 | 0.048 |
| 90 | 5000 | 0.95 | 0.225 |
| 180 | 10000 | 0.95 | 0.449 |
| 365 | 50000 | 0.96 | 0.500 |
| 365 | 50000 | 0.91 | 0.000 |

Note: The last row shows the attestation floor kill switch in action.
