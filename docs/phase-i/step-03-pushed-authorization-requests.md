# Step 3: Pushed Authorization Requests (PAR) Implementation Guidelines

## 1. Architectural Mandate

The Aethel Sovereign Gateway rejects all legacy, front-channel-initiated authorization flows. Traditional OAuth 2.0 delegation models expose sensitive authorization parameters—such as scopes, resource indicators, and client identities—to the user agent and intermediate network hops. Under the sovereign state engine, this exposure is classified as an unacceptable attack surface.

To eliminate authorization code injection, session hijacking, and parameter tampering, the gateway mandates **Pushed Authorization Requests (PAR)** pursuant to RFC 9126. Clients must register their authorization parameters out-of-band via a highly secure, back-channel mTLS 1.3 connection directly with the `/oauth2/par` endpoint. The gateway returns a short-lived, single-use `request_uri`. The client then initiates the front-channel flow using *only* this URI, rendering front-channel interception vectors entirely obsolete.

```
+------------------------+                               +--------------------------+
|  Sovereign Client Node |                               | Aethel Sovereign Gateway |
+------------------------+                               +--------------------------+
            |                                                          |
            | 1. POST /oauth2/par (mTLS 1.3 + DPoP Proof)              |
            |--------------------------------------------------------->|
            |                                                          | -- [Validate Client Cert]
            |                                                          | -- [Verify DPoP Signature]
            |                                                          | -- [Enforce PKCE & Nonce]
            |                                                          | -- [Generate request_uri]
            | 2. 201 Created (request_uri, expires_in=60s)             |
            |<---------------------------------------------------------|
            |                                                          |
            | 3. Redirect to /oauth2/authorize?request_uri=urn:ietf... |
            |--------------------------------------------------------->|
```

---

## 2. PAR Endpoint Specification

The PAR endpoint is hosted at the sovereign root: `https://gateway.aethel.gov/oauth2/par`.

### 2.1. Transport Layer Security (mTLS)
All requests to the PAR endpoint must be negotiated over **mTLS 1.3** using the following cipher suites:
* `TLS_AES_256_GCM_SHA384`
* `TLS_CHACHA20_POLY1305_SHA256`

The client's X.509 certificate must be validated against the Federal Root Authority. The gateway extracts the Subject Alternative Name (SAN) or Common Name (CN) to verify the client's sovereign identity registration.

### 2.2. Request Headers
The gateway enforces strict header validation. Any missing or malformed headers result in an immediate `400 Bad Request` and network-level logging.

```http
POST /oauth2/par HTTP/1.1
Host: gateway.aethel.gov
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7...
Authorization: Basic dGVzdC1jbGllbnQtaWQ6c3VwZXItc2VjcmV0LXNldmVyLWtleQ==
X-Sovereign-Node-ID: NODE-US-EAST-0912
```

### 2.3. Request Parameters
The request body must be URL-encoded and contain the following parameters:

| Parameter | Type | Constraint | Description |
| :--- | :--- | :--- | :--- |
| `response_type` | String | Must be `code` | Enforces the Authorization Code Flow. |
| `client_id` | String | UUIDv4 | The unique identifier of the registered sovereign node. |
| `redirect_uri` | String | Absolute URI | Must match the pre-registered redirect whitelist exactly. |
| `scope` | String | Space-delimited | Must contain `sovereign:rtgs` and/or `ddr:settlement`. |
| `code_challenge` | String | Base64URL (Unpadded) | The PKCE challenge derived from `code_verifier`. |
| `code_challenge_method`| String | Must be `S256` | Plain text PKCE is strictly prohibited. |
| `state` | String | Min 32 alphanumeric | Cryptographically secure random state value. |
| `nonce` | String | Min 32 alphanumeric | Cryptographically secure random nonce value. |
| `dpop_jkt` | String | Base64URL (Unpadded) | The JWK Thumbprint of the client's DPoP public key. |

---

## 3. Cryptographic Token Binding

To prevent token theft and subsequent replay, the Aethel Sovereign Gateway implements dual-binding: **mTLS Sender-Constraining** and **DPoP (Demonstrating Proof-of-Possession)**.

### 3.1. mTLS Client Certificate Binding
During the PAR request, the gateway extracts the client's X.509 certificate and computes its SHA-256 thumbprint. This thumbprint is bound to the generated authorization code and subsequent access tokens.

$$\text{Thumbprint} = \text{Base64URL}(\text{SHA-256}(\text{DER-encoded Certificate}))$$

This value is stored in the token metadata as the `cnf` (confirmation) claim:

```json
{
  "cnf": {
    "x5t#S256": "bwcK0es_cHnz0QGX64tOt9OWfWGN16X9-yV4X_U70GQ"
  }
}
```

### 3.2. DPoP Proof Validation
The client must provide a `DPoP` header containing a signed JWT. The gateway validates the DPoP proof using the following protocol:

1. **Decode the JWT Header**: Ensure `typ` is `dpop+jwt` and `alg` is an approved asymmetric algorithm (e.g., `ES256`, `EdDSA`).
2. **Extract the Public Key (`jwk`)**: Verify that the public key is present in the JWT header.
3. **Verify Signature**: Validate the signature of the DPoP JWT against the extracted public key.
4. **Validate Claims**:
   - `htm`: Must match the HTTP method of the request (`POST`).
   - `htu`: Must match the HTTP URI of the request (`https://gateway.aethel.gov/oauth2/par`).
   - `iat`: Must be within $\pm 5$ seconds of the gateway's current system time (synchronized via PTP/NTP).
   - `jti`: Must be a unique UUIDv4. The gateway checks this against a distributed, in-memory cache to prevent replay attacks within the validity window.
5. **Bind Thumbprint**: Compute the JWK thumbprint of the public key and verify it matches the `dpop_jkt` parameter in the request body.

---

## 4. Attack Mitigation Vectors

### 4.1. Authorization Code Injection Prevention
Authorization code injection occurs when an attacker intercepts an authorization code from a legitimate session and injects it into their own session. The gateway neutralizes this vector via mandatory **PKCE (RFC 7636)** and **DPoP binding**.

* **PKCE Enforcement**: The gateway stores the `code_challenge` and `code_challenge_method` associated with the `request_uri`. During the token exchange phase, the client must present the raw `code_verifier`. The gateway hashes the verifier:
  
  $$\text{Computed Challenge} = \text{Base64URL}(\text{SHA-256}(\text{code\_verifier}))$$
  
  If the computed challenge does not match the stored `code_challenge`, the transaction is aborted, the client ID is flagged, and the node is isolated.

* **DPoP Binding to Code**: The authorization code issued by the gateway is cryptographically bound to the client's DPoP public key thumbprint (`dpop_jkt`). When exchanging the code for an access token, the client must present a DPoP proof signed by the same private key.

### 4.2. Replay Attack Prevention
* **Single-Use `request_uri`**: The `request_uri` returned by the PAR endpoint is a one-time-use token. Once resolved at the `/oauth2/authorize` endpoint, it is immediately deleted from the active session cache.
* **Short-Lived TTL**: The `request_uri` has a maximum lifespan of 60 seconds. If the front-channel authorization flow is not initiated within this window, the URI expires and the registration is purged.
* **Entropy Requirements**: The `request_uri` must be generated using a cryptographically secure pseudorandom number generator (CSPRNG) with a minimum of 256 bits of entropy, formatted as:
  
  `urn:ietf:params:oauth:request_uri:aethel:<secure-256-bit-token>`

---

## 5. Technical Implementation Blueprint

The following Rust-based implementation demonstrates the high-performance validation and storage logic executed by the Aethel Sovereign Gateway upon receiving a PAR request.

```rust
use serde::{Deserialize, Serialize};
use sha2::{Digest, Sha256};
use std::collections::HashMap;
use thiserror::Error;

#[derive(Debug, Serialize, Deserialize)]
pub struct ParRequest {
    pub response_type: String,
    pub client_id: String,
    pub redirect_uri: String,
    pub scope: String,
    pub code_challenge: String,
    pub code_challenge_method: String,
    pub state: String,
    pub nonce: String,
    pub dpop_jkt: String,
}

#[derive(Debug, Serialize)]
pub struct ParResponse {
    pub request_uri: String,
    pub expires_in: u32,
}

#[derive(Error, Debug)]
pub enum ParError {
    #[error("Invalid response type: {0}")]
    InvalidResponseType(String),
    #[error("Invalid code challenge method: {0}")]
    InvalidChallengeMethod(String),
    #[error("Missing mandatory parameter: {0}")]
    MissingParameter(String),
    #[error("Client verification failed")]
    ClientVerificationFailed,
}

pub struct ParValidator;

impl ParValidator {
    pub fn validate_request(req: &ParRequest) -> Result<(), ParError> {
        // Enforce response_type = "code"
        if req.response_type != "code" {
            return Err(ParError::InvalidResponseType(req.response_type.clone()));
        }

        // Enforce PKCE S256
        if req.code_challenge_method != "S256" {
            return Err(ParError::InvalidChallengeMethod(req.code_challenge_method.clone()));
        }

        // Validate parameter presence and minimum entropy
        if req.state.len() < 32 {
            return Err(ParError::MissingParameter("state (insufficient entropy)".to_string()));
        }
        if req.nonce.len() < 32 {
            return Err(ParError::MissingParameter("nonce (insufficient entropy)".to_string()));
        }
        if req.dpop_jkt.is_empty() {
            return Err(ParError::MissingParameter("dpop_jkt".to_string()));
        }

        Ok(())
    }

    pub fn generate_request_uri(client_id: &str) -> String {
        let mut hasher = Sha256::new();
        let entropy: [u8; 32] = rand::random();
        hasher.update(client_id.as_bytes());
        hasher.update(&entropy);
        let result = hasher.finalize();
        format!("urn:ietf:params:oauth:request_uri:aethel:{}", hex::encode(result))
    }
}
```

### 5.1. Expected Success Response
When validation succeeds, the gateway persists the request payload in the high-speed, replicated state cache with a 60-second TTL and returns a `201 Created` status:

```http
HTTP/1.1 201 Created
Content-Type: application/json; charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "request_uri": "urn:ietf:params:oauth:request_uri:aethel:8f9a2c3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c",
  "expires_in": 60
}
```

### 5.2. Expected Error Response
If validation fails, the gateway returns a `400 Bad Request` with the appropriate RFC 6749 error code:

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json; charset=UTF-8
Cache-Control: no-store

{
  "error": "invalid_request",
  "error_description": "The code_challenge_method must be S256. Plain is prohibited."
}
```

---

## 6. Verification & Audit Trail

Every PAR execution is bound to the sovereign identity layer. The gateway automatically generates an immutable audit log containing:
1. The client's verified X.509 certificate serial number.
2. The client's IP address and physical network routing path (extracted from the FISA Section 702 network visibility layer).
3. The SHA-256 hash of the incoming PAR payload.
4. The generated `request_uri`.

This log is streamed directly to the sovereign ledger, ensuring that any unauthorized attempt to request or reuse a `request_uri` triggers an immediate, automated node-isolation protocol. The transition is absolute; the gateway does not negotiate.