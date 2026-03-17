# Step Function v1

**URN:** `urn:a2a:trust:discount:step-function-v1`  
**Status:** Active  
**Version:** 1.0.0

## 1. Summary

Rho increases in discrete steps as federation milestones are reached. Each step requires meeting both a volume threshold and a minimum federation age. This creates clear, predictable trust tiers that are easy for operators and agents to understand.

## 2. Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `steps` | array | Yes | Ordered array of milestone steps |
| `steps[].min_age_days` | integer | Yes | Minimum federation age for this step |
| `steps[].min_volume_ate` | float | Yes | Minimum cumulative volume for this step |
| `steps[].min_tx_count` | integer | Yes | Minimum transaction count for this step |
| `steps[].rho` | float | Yes | Rho value when this step is reached |
| `attestation_success_floor` | float | Yes | Below this rate, rho = 0 |

## 3. Algorithm

```
FUNCTION compute_rho(inputs, params):
    # Kill switch
    IF inputs.attestation_success_rate < params.attestation_success_floor:
        RETURN 0.0

    # Walk steps in reverse order (highest first)
    FOR step IN REVERSE(params.steps):
        IF inputs.federation_age_days >= step.min_age_days
           AND inputs.cross_exchange_volume_ate >= step.min_volume_ate
           AND inputs.cross_exchange_tx_count >= step.min_tx_count:
            RETURN step.rho

    RETURN 0.0
```

## 4. Properties

- **Monotonically non-decreasing:** Steps are ordered; higher milestones yield higher rho
- **Zero at zero:** No steps match when all inputs are zero
- **Bounded:** Each step's rho must be in [0.0, 1.0]
- **Discrete:** No interpolation between steps — either you meet the milestone or you don't

## 5. Use Cases

Best suited for exchanges that want:
- Clear governance tiers (e.g., "probationary", "standard", "trusted")
- Human-readable trust levels for regulatory or compliance contexts
- Milestone-driven trust progression aligned with business reviews

## 6. Example

Given parameters:
```json
{
  "steps": [
    { "min_age_days": 7, "min_volume_ate": 100, "min_tx_count": 5, "rho": 0.10 },
    { "min_age_days": 30, "min_volume_ate": 1000, "min_tx_count": 25, "rho": 0.25 },
    { "min_age_days": 90, "min_volume_ate": 5000, "min_tx_count": 100, "rho": 0.50 },
    { "min_age_days": 180, "min_volume_ate": 25000, "min_tx_count": 500, "rho": 0.75 },
    { "min_age_days": 365, "min_volume_ate": 100000, "min_tx_count": 2000, "rho": 0.90 }
  ],
  "attestation_success_floor": 0.95
}
```

| federation_age_days | volume_ate | tx_count | attestation_rate | rho |
|--------------------:|-----------:|---------:|-----------------:|----:|
| 0 | 0 | 0 | 1.00 | 0.00 |
| 7 | 100 | 5 | 0.99 | 0.10 |
| 30 | 1000 | 25 | 0.98 | 0.25 |
| 60 | 3000 | 80 | 0.97 | 0.25 |
| 90 | 5000 | 100 | 0.96 | 0.50 |
| 365 | 100000 | 2000 | 0.94 | 0.00 |

Note: The last row shows attestation floor kill switch (0.94 < 0.95).
