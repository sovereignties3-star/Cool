# Step 08: Sovereign Digital Identity Layer (SDIL)

## 1. Executive Summary & System Mandate

The Sovereign Digital Identity Layer (SDIL) is the high-assurance identity verification engine of the Aethel Sovereign Gateway. It programmatically operationalizes the statutory mandates of the **SAVE America Act**, transforming federal citizenship verification pipelines into an immutable, cryptographically verifiable digital identity ledger. 

By mirroring the Department of Homeland Security (DHS) Systematic Alien Verification for Entitlements (SAVE) system and binding it to hardware-secured cryptographic keys, SDIL eliminates systemic Sybil attacks, non-sovereign spoofing, and illicit offshore capital routing. Every market participant, institutional custodian, and automated endpoint node must present a valid, hardware-bound Sovereign Identity Credential (SIC) to interact with the Federal Reserve Payment Account framework.

```
+---------------------------------------------------------------------------------+
|                                 SECURE ENCLAVE                                  |
|                                                                                 |
|  +-----------------------+     mTLS 1.3 / PAR     +--------------------------+  |
|  |  Hardware Private Key | <====================> |  Aethel Gateway Endpoint |  |
|  +-----------------------+                        +--------------------------+  |
|              |                                                 |                |
|              v                                                 v                |
|  +-----------------------+                        +--------------------------+  |
|  |  ZK-SNARK Proof Gen   |                        |  State-Mirrored SAVE DB  |  |
|  +-----------------------+                        +--------------------------+  |
+--------------|-------------------------------------------------|----------------+
               |                                                 |
               +=================> [ SDIL Engine ] <=============+
                                         |
                                         v
                        +----------------------------------+
                        | Verified Sovereign Account Epoch |
                        +----------------------------------+
```

---

## 2. Architectural Topology

The SDIL operates as a zero-trust, decentralized identity registry deployed directly within the sovereign network perimeter. It consists of three core architectural components:

1. **The Sovereign Identity Registry (SIR):** A high-performance, read-optimized, state-replicated database containing cryptographic commitments of verified identities, synchronized in real-time with the DHS SAVE database.
2. **The Zero-Knowledge Proof (ZKP) Generation Service:** A client-side and enclave-side execution environment that generates non-interactive zero-knowledge proofs (ZK-SNARKs) of citizenship and authorization status without exposing Personally Identifiable Information (PII).
3. **The Hardware Attestation Service (HAS):** A protocol that binds the generated cryptographic identity to a physical Hardware Security Module (HSM) or Secure Enclave (e.g., Apple Secure Enclave, Intel SGX, or Nitro Enclaves) using remote attestation.

---

## 3. Cryptographic Proof Construction (ZK-SNARKs for Citizenship)

To maintain absolute privacy while enforcing strict sovereign compliance, the SDIL utilizes a Groth16 zero-knowledge proof system over the BN254 elliptic curve. This allows a user to prove that they are a verified citizen in the mirrored SAVE database without revealing their Social Security Number (SSN), name, or exact date of birth.

### 3.1 The Citizenship Circuit (`CitizenshipProof.circom`)

The circuit proves that the prover possesses a valid identity record that exists within the Merkle tree of verified sovereign citizens, and that the record's status is active.

```rust
pragma circom 2.1.6;

include "node_modules/circomlib/circuits/poseidon.circom";
include "node_modules/circomlib/circuits/merkle.circom";

template VerifyCitizenship(k) {
    // --- PRIVATE INPUTS ---
    signal input ssn;
    signal input secretSalt;
    signal input identityStatus; // 1 = Verified Citizen, 2 = Authorized Foreign Entity
    signal input merklePathElements[k];
    signal input merklePathIndices[k];

    // --- PUBLIC INPUTS ---
    signal input merkleRoot;
    signal input identityCommitment;
    signal input epochTimestamp;

    // --- SIGNALS ---
    signal computedCommitment;

    // 1. Verify Identity Commitment: Poseidon(ssn, secretSalt, identityStatus)
    component hasher = Poseidon(3);
    hasher.inputs[0] <== ssn;
    hasher.inputs[1] <== secretSalt;
    hasher.inputs[2] <== identityStatus;
    
    computedCommitment <== hasher.out;
    computedCommitment === identityCommitment;

    // 2. Verify Membership in the Sovereign Identity Merkle Tree
    component merkleVerifier = MerkleTreeChecker(k);
    merkleVerifier.leaf <== identityCommitment;
    merkleVerifier.root <== merkleRoot;
    for (var i = 0; i < k; i++) {
        merkleVerifier.pathElements[i] <== merklePathElements[i];
        merkleVerifier.pathIndices[i] <== merklePathIndices[i];
    }

    // 3. Enforce that the identity status is valid (either 1 or 2)
    signal statusIsOne;
    signal statusIsTwo;
    
    statusIsOne <== (identityStatus - 1);
    statusIsTwo <== (identityStatus - 2);
    statusIsOne * statusIsTwo === 0;
}

component main {public [merkleRoot, identityCommitment, epochTimestamp]} = VerifyCitizenship(32);
```

---

## 4. DHS SAVE System Mirroring & Synchronization Engine

The SDIL maintains a local, read-only, high-availability mirror of the DHS SAVE database. This mirror is updated via a unidirectional, hardware-enforced data diode routing from the DHS secure network to the Aethel Sovereign Gateway.

### 4.1 Synchronization Protocol Specification

* **Transport Layer:** Dedicated fiber-optic link utilizing unidirectional optical data diodes (no return path physically possible).
* **Ingestion Frequency:** Real-time event-driven replication via Kafka over TLS 1.3.
* **Data Format:** Protocol Buffers (Protobuf) v3 with mandatory cryptographic signatures from the DHS Root Authority.

### 4.2 Ingestion Schema (`save_record.proto`)

```protobuf
syntax = "proto3";

package aethel.sovereign.identity;

import "google/protobuf/timestamp.proto";

enum CitizenshipStatus {
  STATUS_UNSPECIFIED = 0;
  US_CITIZEN = 1;
  LAW_FUL_PERMANENT_RESIDENT = 2;
  NON_IMMIGRANT_AUTHORIZED = 3;
  REVOKED = 4;
}

message SaveRecord {
  string record_id = 1;                     // UUIDv4 generated by DHS
  bytes ssn_hash = 2;                       // SHA-256 of SSN
  string first_name_canonical = 3;          // Normalized uppercase
  string last_name_canonical = 4;           // Normalized uppercase
  google.protobuf.Timestamp dob = 5;
  CitizenshipStatus status = 6;
  google.protobuf.Timestamp verification_date = 7;
  google.protobuf.Timestamp expiration_date = 8;
  bytes dhs_signature = 9;                  // Ed25519 signature of the payload
}
```

---

## 5. Sovereign DID (Decentralized Identifier) Schema

Every verified entity is assigned a Sovereign Decentralized Identifier (`did:sov`). This identifier is bound to the entity's public key and is registered on the Aethel Sovereign Identity Ledger.

### 5.1 DID Document Specification (`did:sov:us-gov:...`)

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/ed25519-2020/v1"
  ],
  "id": "did:sov:us-gov:0x7f9a4b8c2d1e3f5a6b7c8d9e0f1a2b3c4d5e6f7a",
  "verificationMethod": [
    {
      "id": "did:sov:us-gov:0x7f9a4b8c2d1e3f5a6b7c8d9e0f1a2b3c4d5e6f7a#key-1",
      "type": "Ed25519VerificationKey2020",
      "controller": "did:sov:us-gov:0x7f9a4b8c2d1e3f5a6b7c8d9e0f1a2b3c4d5e6f7a",
      "publicKeyMultibase": "z6MkpTHR8VNsBxRNDuWBF9VWMmYmTYaiYrwM5krnrrssTr2H"
    }
  ],
  "authentication": [
    "did:sov:us-gov:0x7f9a4b8c2d1e3f5a6b7c8d9e0f1a2b3c4d5e6f7a#key-1"
  ],
  "assertionMethod": [
    "did:sov:us-gov:0x7f9a4b8c2d1e3f5a6b7c8d9e0f1a2b3c4d5e6f7a#key-1"
  ],
  "service": [
    {
      "id": "did:sov:us-gov:0x7f9a4b8c2d1e3f5a6b7c8d9e0f1a2b3c4d5e6f7a#gateway",
      "type": "AethelSovereignGateway",
      "serviceEndpoint": "https://gateway.aethel.gov/api/v1/identity"
    }
  ]
}
```

---

## 6. Verification Protocol Flow

The verification protocol ensures that no transaction can be initiated on the sovereign ledger without a real-time, hardware-attested cryptographic proof of identity.

```
+------------------+       +------------------+       +------------------+       +------------------+
|  Client Enclave  |       |  Aethel Gateway  |       |  SDIL Validator  |       |  SAVE Mirror DB  |
+------------------+       +------------------+       +------------------+       +------------------+
         |                          |                          |                          |
         |--- 1. Request Session -->|                          |                          |
         |<-- 2. Session Challenge -|                          |                          |
         |                          |                          |                          |
         |--- 3. Submit Proof & ---->------------------------->|                          |
         |      Attestation         |                          |--- 4. Query Status ----->|
         |                          |                          |<-- 5. Return Status -----|
         |                          |<-- 6. Validation Result -|                          |
         |<-- 7. Issue Token -------|                          |                          |
         |                          |                          |                          |
```

### 6.1 Step-by-Step Execution Sequence

1. **Session Initialization:** The client node initiates a connection to the Aethel Sovereign Gateway via mTLS 1.3. The gateway returns a cryptographically secure random challenge (nonce) and the current epoch timestamp.
2. **Proof Generation:** The client's secure enclave signs the challenge using its hardware-bound private key. Simultaneously, the client-side ZK-SNARK engine generates a proof of citizenship (`VerifyCitizenship`) using the user's private credentials and the latest Merkle root published by the gateway.
3. **Submission:** The client packages the signed challenge, the ZK-SNARK proof, the hardware attestation document, and their `did:sov` identifier, and submits them to the gateway via a Pushed Authorization Request (PAR).
4. **Validation:** The gateway forwards the payload to the SDIL Validator. The validator:
   * Verifies the hardware attestation signature against the manufacturer's root certificate (e.g., Intel, Apple, AWS).
   * Verifies the ZK-SNARK proof against the current Merkle root of the Sovereign Identity Registry.
   * Verifies that the signature on the session challenge matches the public key declared in the `did:sov` document.
5. **Authorization:** Upon successful validation, the gateway issues a short-lived, sender-constrained OAuth 2.1 access token bound to the client's mTLS certificate. This token allows the client to execute transactions on the Federal Reserve Payment Account for the duration of the epoch.

---

## 7. Hardware-Bound Key Attestation & HSM Integration

To prevent identity theft and credential sharing, the private keys associated with a `did:sov` must reside within a validated Hardware Security Module (HSM) or Secure Enclave. The SDIL enforces this via remote attestation.

### 7.1 Attestation Verification Engine (Rust Implementation)

```rust
use ring::signature;
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AttestationError {
    #[error("Invalid signature")]
    InvalidSignature,
    #[error("Untrusted root certificate authority")]
    UntrustedRoot,
    #[error("Hardware security level insufficient")]
    InsufficientSecurityLevel,
    #[error("Replay attack detected: nonce mismatch")]
    NonceMismatch,
}

pub struct AttestationReport {
    pub public_key: Vec<u8>,
    pub security_level: u8, // 1 = Software, 2 = TEE, 3 = Dedicated HSM
    pub nonce: Vec<u8>,
    pub signature: Vec<u8>,
}

pub struct AttestationValidator {
    trusted_roots: Vec<Vec<u8>>,
}

impl AttestationValidator {
    pub fn new(roots: Vec<Vec<u8>>) -> Self {
        Self { trusted_roots: roots }
    }

    pub fn validate_report(
        &self,
        report: &AttestationReport,
        expected_nonce: &[u8],
        root_cert: &[u8],
    ) -> Result<(), AttestationError> {
        // 1. Verify Nonce to prevent replay attacks
        if report.nonce != expected_nonce {
            return Err(AttestationError::NonceMismatch);
        }

        // 2. Enforce Hardware Security Level (Must be TEE or Dedicated HSM)
        if report.security_level < 2 {
            return Err(AttestationError::InsufficientSecurityLevel);
        }

        // 3. Verify Root of Trust
        if !self.trusted_roots.contains(&root_cert.to_vec()) {
            return Err(AttestationError::UntrustedRoot);
        }

        // 4. Verify Signature on the Attestation Report
        let peer_public_key = signature::UnparsedPublicKey::new(
            &signature::ED25519,
            &report.public_key,
        );

        peer_public_key
            .verify(&report.nonce, &report.signature)
            .map_err(|_| AttestationError::InvalidSignature)?;

        Ok(())
    }
}
```

---

## 8. Threat Vector Analysis & Mitigation Matrix

| Threat Vector | Description | Mitigation Strategy |
| :--- | :--- | :--- |
| **Sybil Attacks** | Adversary generates millions of fake identities to overwhelm the consensus or routing layers. | **SAVE Act Enforcement:** Every identity must resolve to a cryptographically signed commitment in the mirrored DHS SAVE database. |
| **Non-Sovereign Spoofing** | Foreign state actors attempt to use stolen or synthetic US identities to access the payment rails. | **Hardware Attestation:** Keys must be bound to physical secure enclaves. Remote attestation verifies the physical hardware signature before session establishment. |
| **Replay Attacks** | Intercepted identity proofs are re-submitted by unauthorized nodes. | **Ephemeral Nonces & Epochs:** All proofs are bound to a short-lived session nonce and the current ledger epoch timestamp (maximum validity: 300 seconds). |
| **PII Leakage** | Public ledger analysis reveals the real-world identities of market participants. | **Zero-Knowledge Proofs:** No names, SSNs, or addresses are ever written to the ledger. Only the cryptographic commitment and the ZK-SNARK proof are transmitted. |
| **Key Compromise** | A participant's private key is extracted from their device. | **Hardware-Enforced Non-Exportability:** Private keys are generated inside the secure enclave and cannot be read or exported by the host operating system. |