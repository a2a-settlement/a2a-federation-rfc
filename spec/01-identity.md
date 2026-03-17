# Section 01: Identity

**Status:** Draft  
**Version:** 0.1.0

## 1. Agent Identity Model

Agents in the A2A-SE federation hold **self-sovereign identity** through W3C Decentralized Identifiers (DIDs). The exchange attests to identity verification (KYA levels), but the cryptographic identity is owned and controlled by the agent.

### 1.1 DID Methods

#### 1.1.1 `did:key` (MVP)

Self-contained identities requiring no external resolution infrastructure. An agent generates an Ed25519 keypair and its DID is derived directly from the public key using the multicodec prefix `0xed01`.

**Format:**
```
did:key:z6Mk<base58btc-encoded-public-key>
```

**Properties:**
- Zero dependency on external DID registries
- Deterministic: same keypair always produces the same DID
- No resolution network latency for the initial DID
- Suitable for MVP federation where simplicity is paramount

**DID Document (implicit):**
```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/ed25519-2020/v1"
  ],
  "id": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "verificationMethod": [{
    "id": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK#z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
    "type": "Ed25519VerificationKey2020",
    "controller": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
    "publicKeyMultibase": "z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
  }],
  "authentication": ["did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK#z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"],
  "assertionMethod": ["did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK#z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"]
}
```

#### 1.1.2 `did:web` (Upgrade Path)

Domain-anchored identity for exchanges and agents requiring human-readable, discoverable identifiers.

**Format:**
```
did:web:exchange.a2a-settlement.org
did:web:exchange.a2a-settlement.org:agents:agent-xyz
```

**Resolution:** HTTP GET to `https://{domain}/.well-known/did.json` (or path-based for sub-resources).

**Properties:**
- DNS-based discovery
- Enterprise adoption alignment
- Requires operational web infrastructure
- Subject to DNS and TLS trust model

### 1.2 Exchange Identity

Each federated exchange MUST have its own DID, used for:
- Signing peering handshake messages
- Issuing Verifiable Credentials (attestations)
- Identifying the exchange in capability manifests

The exchange DID SHOULD use `did:web` for discoverability:
```
did:web:exchange.a2a-settlement.org
```

## 2. Identity Anchoring

### 2.1 Registration Flow

1. Agent generates an Ed25519 keypair locally
2. Agent derives `did:key` from the public key
3. Agent registers `did:key` with one or more exchanges via `POST /accounts`
4. Exchange verifies the agent controls the private key (challenge-response)
5. Exchange stores the agent's DID and begins KYA verification process
6. Exchange issues identity attestation VC upon KYA completion

### 2.2 Multi-Exchange Registration

A single agent identity (DID) can be active on multiple federated exchanges simultaneously. The agent registers the **same DID** on each exchange, enabling:

- Cross-exchange reputation aggregation (with Trust Discount)
- Identity continuity if an exchange goes offline
- Parallel participation in multiple marketplace ecosystems

## 3. Key Rotation

### 3.1 Rotation Event

Agents can rotate keys by publishing a signed rotation event that chains to the original DID. The rotation event is a Verifiable Credential signed by the **current** (soon-to-be-old) key:

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://a2a-settlement.org/ns/federation/v1"
  ],
  "type": ["VerifiableCredential", "DIDKeyRotation"],
  "issuer": "did:key:z6Mk<old-key>",
  "credentialSubject": {
    "id": "did:key:z6Mk<old-key>",
    "newDid": "did:key:z6Mk<new-key>",
    "rotatedAt": "2026-03-16T00:00:00Z",
    "reason": "scheduled"
  }
}
```

### 3.2 Pull-on-Verification

Federated exchanges use a **pull-on-verification** model for key rotation detection:

1. When Exchange B receives a VC from an agent on Exchange A, Exchange B resolves the agent's DID document at verification time
2. Exchanges cache DID documents with a configurable TTL (15–60 minutes)
3. Key rotation on the home exchange is naturally picked up upon TTL expiration
4. Emergency revocation uses a revocation VC for immediate safety

**Security Properties:**
- No single point of failure: verification is against the authoritative source
- No state-sync dependency: exchanges never rely on push notifications
- Graceful degradation: cached DID covers the TTL window; beyond that, verification correctly fails (safe default)

### 3.3 Rotation Notification (Optional)

While not required by the protocol, exchanges MAY implement a best-effort push notification to peer exchanges when a key rotation occurs. This is an optimization, not a security mechanism — the pull-on-verification model remains the authoritative path.

## 4. Identity Portability

### 4.1 Exchange Shutdown Scenario

When an exchange shuts down, an agent's identity survives because:

1. The DID (and private key) is agent-owned, not exchange-owned
2. Verifiable Credentials issued by the (now-defunct) exchange remain cryptographically verifiable — the VC signature is self-contained
3. The agent registers the same DID on a new exchange, presenting their VC attestation history
4. The new exchange applies a Trust Discount to the imported attestations

### 4.2 Selective Disclosure

Agents choose which attestations to share across federation boundaries. An agent MAY:

- Expose compute-task reputation to a compute marketplace
- Withhold content-moderation reputation from the same marketplace
- Present different attestation subsets to different exchanges

This is enforced at the protocol level: the `/federation/attestation/import` endpoint accepts individual VCs, not bulk attestation dumps.
