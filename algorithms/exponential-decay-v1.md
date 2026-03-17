# Exponential Decay v1

**URN:** `urn:a2a:trust:discount:exponential-decay-v1`  
**Status:** Active  
**Version:** 1.0.0

## 1. Summary

Rho approaches `max_rho` asymptotically with diminishing returns on additional volume. Early volume has a high marginal impact on trust; later volume has diminishing returns. This creates a natural "trust plateau" where exchanges earn most of their trust quickly but reaching full trust requires sustained, long-term activity.

## 2. Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `max_rho` | float | Yes | Asymptotic maximum rho |
| `volume_half_life_ate` | float | Yes | ATE volume at which rho reaches 50% of max_rho |
| `age_weight` | float | No | Weight factor for federation age component (default: 0.1) |
| `attestation_success_floor` | float | Yes | Below this rate, rho = 0 |

## 3. Algorithm

```
FUNCTION compute_rho(inputs, params):
    # Kill switch
    IF inputs.attestation_success_rate < params.attestation_success_floor:
        RETURN 0.0

    # Age component (small, linear, capped at age_weight)
    age_weight = params.age_weight OR 0.1
    age_component = MIN(inputs.federation_age_days / 365.0, 1.0) * age_weight

    # Volume component (exponential approach to max_rho - age_weight)
    volume_ceiling = params.max_rho - age_weight
    IF inputs.cross_exchange_volume_ate <= 0:
        volume_component = 0.0
    ELSE:
        # 1 - e^(-k*v) where k = ln(2) / half_life
        k = LN(2) / params.volume_half_life_ate
        volume_component = volume_ceiling * (1.0 - EXP(-k * inputs.cross_exchange_volume_ate))

    RETURN MIN(age_component + volume_component, params.max_rho)
```

## 4. Properties

- **Monotonically non-decreasing:** Exponential approach guarantees this
- **Zero at zero:** `1 - e^0 = 0` and `0/365 * 0.1 = 0`
- **Bounded:** Asymptotically approaches `max_rho`, never exceeds it
- **Diminishing returns:** First 50% of trust is cheapest; last 10% requires massive volume
- **Half-life intuition:** At `volume_half_life_ate` volume, rho is approximately 50% of its volume ceiling

## 5. Use Cases

Best suited for exchanges that want:
- Quick initial trust building (good for bootstrapping federation)
- Natural ceiling that prevents any single peer from reaching full parity
- Economic disincentive for trust-splitting attacks (splitting volume across fake exchanges reduces rho)

## 6. Example

Given parameters:
```json
{
  "max_rho": 0.85,
  "volume_half_life_ate": 5000,
  "age_weight": 0.10,
  "attestation_success_floor": 0.90
}
```

| federation_age_days | cross_exchange_volume_ate | attestation_success_rate | rho |
|--------------------:|-------------------------:|-------------------------:|------:|
| 0 | 0 | 1.00 | 0.000 |
| 30 | 1000 | 0.98 | 0.106 |
| 90 | 5000 | 0.95 | 0.400 |
| 180 | 15000 | 0.95 | 0.709 |
| 365 | 50000 | 0.96 | 0.850 |
| 365 | 50000 | 0.89 | 0.000 |

Note: At the half-life volume (5000 ATE), the volume component is approximately half of `volume_ceiling` (0.75), yielding ~0.375 from volume plus the age component.
