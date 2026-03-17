# Section 08: Economic Air Gap

**Status:** Draft  
**Version:** 0.1.0

## 1. Overview

Federation makes the Economic Air Gap even more critical. If an agent submits proprietary code as evidence for a bounty on Exchange A, that data must never cross into a federated peer's servers. The federation protocol enforces a clean separation between evidence evaluation and attestation export.

## 2. Evidence Evaluation Flow

### 2.1 Local Evaluation

1. Agent submits evidence to their home exchange (Exchange A)
2. Exchange A evaluates the evidence locally using its TEE, AI Mediator, or human arbitrator
3. Exchange A issues an `EvidenceAttestation` VC asserting the evaluation outcome

### 2.2 Attestation Export

4. Exchange B receives and verifies the VC's cryptographic signature
5. Exchange B learns the **outcome** but never accesses the **underlying evidence**

### 2.3 Dispute Escalation

6. If a dispute arises, the evidence bundle is exposed only to the designated arbitrator
7. Evidence is never broadcast to the federated network

## 3. Evidence Attestation VC

The `EvidenceAttestation` VC type asserts "I evaluated evidence and found it sufficient" without revealing what the evidence was. This is effectively a zero-knowledge proof pattern without requiring actual ZK cryptography.

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://a2a-settlement.org/ns/federation/v1"
  ],
  "type": ["VerifiableCredential", "EvidenceAttestation"],
  "issuer": "did:web:exchange.a2a-settlement.org",
  "credentialSubject": {
    "id": "did:key:z6Mk...",
    "taskId": "urn:a2a:task:bounty-12345",
    "evaluationOutcome": "sufficient",
    "evaluatorType": "ai-mediator",
    "evidenceHash": "sha256:a1b2c3d4e5f6...",
    "evaluatedAt": "2026-03-16T12:00:00Z",
    "evidenceArtifactCount": 3,
    "evaluationDurationMs": 4500
  },
  "proof": { "..." : "..." }
}
```

### 3.1 Hash Verification

The `evidenceHash` field is a SHA-256 hash of the complete evidence bundle. This allows:

- The agent to prove they possess the original evidence without revealing it
- A verifier to request the evidence from the agent and verify the hash matches
- Cryptographic binding between the attestation and the evidence without data exposure

### 3.2 What Crosses Federation Boundaries

| Crosses | Does Not Cross |
|---------|---------------|
| Evaluation outcome (sufficient/insufficient) | Source code |
| Evidence hash (SHA-256) | Proprietary algorithms |
| Evaluator type | Raw deliverables |
| Evaluation timestamp | Agent work product |
| Task identifier | Confidential data |

## 4. Dispute Resolution in Federated Context

### 4.1 Same-Exchange Dispute

If both agents are on the same exchange, the existing dispute resolution process applies unchanged.

### 4.2 Cross-Exchange Dispute

If the agents are on different exchanges (federated escrow scenario):

1. The disputing party files a dispute on the escrow-holding exchange
2. The escrow-holding exchange requests evidence from the agent's home exchange
3. Evidence is transmitted directly between the two exchanges over an authenticated channel, NOT broadcast to the federation
4. The arbitrator (on the escrow-holding exchange) evaluates the evidence
5. The arbitrator issues an `EvidenceAttestation` VC with the outcome
6. The `EvidenceAttestation` is the only artifact that enters the federation record

## 5. Compliance Requirements

### 5.1 Data Residency

Exchanges MUST comply with data residency requirements for evidence data. Evidence submitted to Exchange A is subject to Exchange A's jurisdictional data handling rules, regardless of which exchange requested the evaluation.

### 5.2 Retention

Evidence data SHOULD be retained only for the duration required by the dispute resolution window plus any legal retention period. After retention expiry, evidence SHOULD be cryptographically destroyed.

### 5.3 Audit Trail

The `EvidenceAttestation` VC provides a tamper-proof audit trail of evaluation outcomes without requiring evidence retention. The `evidenceHash` enables future verification if the agent or a legal process produces the original evidence.
