# Section 06: Federation Health

**Status:** Draft  
**Version:** 0.1.0

## 1. Overview

Uptime is a trust metric. The federation protocol mandates a standardized health endpoint that enables peer exchanges to monitor each other's operational status and automatically adjust trust levels based on reliability.

## 2. Health Endpoint

```
GET /.well-known/a2a-federation-health
```

### 2.1 Response Payload

```json
{
  "status": "operational",
  "node_did": "did:web:exchange.a2a-settlement.org",
  "avg_attestation_latency_ms": 45,
  "uptime_90d": 0.9987,
  "version": "0.1.0",
  "timestamp": "2026-03-16T12:00:00Z",
  "active_peers": 3,
  "federation_protocol_version": "0.1.0"
}
```

### 2.2 Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | enum | Yes | `operational`, `degraded`, or `maintenance` |
| `node_did` | string | Yes | The exchange's DID |
| `avg_attestation_latency_ms` | integer | Yes | Rolling average verification engine speed |
| `uptime_90d` | float | Yes | 90-day rolling uptime percentage |
| `version` | string | Yes | Exchange software version |
| `timestamp` | string | Yes | Current server time (ISO 8601) |
| `active_peers` | integer | Recommended | Number of active federation peers |
| `federation_protocol_version` | string | Yes | Supported federation protocol version |

### 2.3 Status Semantics

| Status | Meaning | Trust Impact |
|--------|---------|-------------|
| `operational` | All systems nominal | None — ρ unchanged |
| `degraded` | Partial service disruption | ρ may be reduced by peers |
| `maintenance` | Planned downtime | ρ decay paused if pre-announced |

## 3. Automatic Trust Decay

### 3.1 Trigger

If a peer fails to respond to health checks, the monitoring exchange automatically triggers exponential decay on the peer's Trust Discount ρ multiplier.

**Parameters:**
- Health check interval: configurable per peer, recommended default **5 minutes**
- Decay trigger: **3 consecutive** failed health checks
- Decay rate: exponential — `ρ_new = ρ_current × 0.9` per missed interval
- Decay floor: exchange's configured quarantine threshold (typically 0.0)

### 3.2 Recovery

Once health checks resume, ρ recovers at a **slower linear rate** (asymmetric recovery). This prevents gaming by briefly restoring service then going down again.

**Recovery formula:**
```
ρ_recovery = ρ_current + (recovery_increment × consecutive_successful_checks)
```

Where `recovery_increment` is typically 0.01–0.05 per successful check interval.

### 3.3 Pre-Announced Maintenance

Exchanges MAY announce planned maintenance by setting `status: "maintenance"` before the downtime begins. Peers that observe the maintenance status SHOULD pause ρ decay for the duration, up to a configurable maximum (e.g., 24 hours).

## 4. Health Monitoring Integration

### 4.1 Trust Discount Input

While not a mandatory Trust Discount input (to keep the MVP interface clean), the RFC RECOMMENDS that exchanges factor `uptime_90d` and `avg_attestation_latency_ms` into their ρ calculation.

Exchanges with aggressive discount curves will naturally weight these heavily; forgiving exchanges may treat them as tiebreakers.

### 4.2 Alerting

Exchanges SHOULD implement alerting when:
- A peer's `uptime_90d` drops below a configurable threshold
- A peer's `avg_attestation_latency_ms` exceeds a configurable maximum
- A peer's `status` changes from `operational` to `degraded` or `maintenance`
- ρ decay triggers for any peer

## 5. Security Considerations

### 5.1 Health Endpoint Authentication

The health endpoint is public (no authentication required) to enable monitoring by any interested party. The endpoint MUST NOT expose sensitive operational data beyond the fields defined in this specification.

### 5.2 Rate Limiting

Exchanges SHOULD rate-limit the health endpoint to prevent abuse. A recommended limit is 60 requests per minute per IP.

### 5.3 Spoofing

The health endpoint response is informational and does not carry a cryptographic proof. Peers SHOULD verify health claims by correlating with their own observation of the peer's behavior (e.g., response latency on `/federation/verify` requests).
