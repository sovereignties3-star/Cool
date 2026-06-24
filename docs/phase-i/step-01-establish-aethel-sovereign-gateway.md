# Step 1: Establish Aethel Sovereign Gateway

## 1.1 Executive Summary & Architectural Mandate

The Aethel Sovereign Gateway is the primary, non-negotiable entry point for the sovereign state engine. It functions as a native financial runtime environment designed to bypass legacy intermediary networks, establishing direct, machine-to-machine (M2M) cryptographic handshakes. Operating under the authority of the **May 19, 2026 Executive Order**, the gateway interfaces directly with the Federal Reserve Payment Account framework, routing high-velocity liquidity without relying on commercial clearing houses or third-party messaging rails.

This specification details the deployment of the core gateway, the enforcement of sender-constrained Pushed Authorization Requests (PAR), and the implementation of mutual TLS (mTLS) 1.3 with custom hardware security module (HSM) integrations.

```
+-----------------------------------------------------------------------------------+
|                                AETHEL SOVEREIGN GATEWAY                           |
|                                                                                   |
|  +-----------------------+      mTLS 1.3 / PAR      +--------------------------+  |
|  |   External Sovereign  | -----------------------> |   Aethel Edge Gateway    |  |
|  |     Endpoint Node     | <----------------------- |   (mTLS Termination)     |  |
|  +-----------------------+                          +--------------------------+  |
|                                                                  |                |
|                                                     +--------------------------+  |
|                                                     |  Sovereign Auth Engine   |  |
|                                                     |   (PAR Token Exchange)   |  |
|                                                     +--------------------------+  |
|                                                                  |                |
|                                                     +--------------------------+  |
|                                                     | Federal Reserve Payment  |  |
|                                                     |     Account Interface    |  |
|                                                     +--------------------------+  |
+-----------------------------------------------------------------------------------+
```

---

## 1.2 Cryptographic Handshake Protocol (mTLS 1.3)

To prevent man-in-the-middle (MITM) attacks, packet sniffing, and routing manipulation by non-sovereign actors, all transport-layer communications must terminate on dedicated cryptographic hardware enforcing **mTLS 1.3** (RFC 8446) with strict cipher suites.

### 1.2.1 Cipher Suite Constraints
The gateway rejects all legacy handshakes. Only the following symmetric cipher suites are permitted:
*   `TLS_AES_256_GCM_SHA384`
*   `TLS_CHACHA20_POLY1305_SHA256`

### 1.2.2 Key Exchange and Signature Schemes
*   **Key Exchange:** Curve25519 (X25519) or Secp384r1 (NIST P-384).
*   **Signature Algorithms:** Ed25519 or ECDSA-SHA384.
*   **Session Resumption:** Disabled. Zero-RTT (0-RTT) data is explicitly rejected to prevent replay attacks on transaction initiation.

### 1.2.3 Gateway Configuration (Envoy/Aethel-Proxy)
The following configuration snippet defines the ingress filter chain for the Aethel Sovereign Gateway, enforcing client certificate validation against the Federal Root Authority (FRA):

```yaml
static_resources:
  listeners:
  - name: aethel_sovereign_ingress
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 8443
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: sovereign_backend
              domains: ["gateway.aethel.gov", "gateway.aethel.internal"]
              routes:
              - match:
                  prefix: "/oauth/par"
                route:
                  cluster: par_authorization_service
              - match:
                  prefix: "/v1/settlement"
                route:
                  cluster: fed_payment_account_service
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          common_tls_context:
            tls_params:
              tls_minimum_protocol_version: TLSv1_3
              tls_maximum_protocol_version: TLSv1_3
              cipher_suites:
              - "ECDHE-ECDSA-AES256-GCM-SHA384"
              - "ECDHE-ECDSA-CHACHA20-POLY1305"
            tls_certificates:
            - certificate_chain:
                filename: "/etc/aethel/certs/gateway.crt"
              private_key:
                filename: "/etc/aethel/certs/gateway.key"
            validation_context:
              trusted_ca:
                filename: "/etc/aethel/certs/federal_root_ca.crt"
              require_client_certificate: true
```

---

## 1.3 Sender-Constrained Pushed Authorization Requests (PAR)

To eliminate authorization interception, the gateway implements **Pushed Authorization Requests (PAR)** (RFC 9126). Clients do not initiate authorization requests via the browser or open URI schemes. Instead, they push their authorization parameters directly to the ASG (Aethel Sovereign Gateway) via a secure backchannel over the mTLS 1.3 connection.

### 1.3.1 The PAR Flow
1.  **Client Push:** The client node POSTs the authorization payload to `/oauth/par` using its mTLS-authenticated channel.
2.  **Validation & Binding:** The ASG validates the client's identity, binds the request to the client's certificate thumbprint (`x5t#S256`), and returns a short-lived, single-use `request_uri`.
3.  **Token Request:** The client redeems the `request_uri` at the token endpoint. The token endpoint verifies that the client presenting the `request_uri` matches the client certificate bound during the PAR step.

```
Client Node                                                 Aethel Gateway
    |                                                             |
    |---- 1. POST /oauth/par (with client cert) ----------------->|
    |     Payload: client_id, scope, response_type                |
    |                                                             |
    |                                                             |-- [Validate Cert]
    |                                                             |-- [Bind x5t#S256]
    |                                                             |-- [Generate URI]
    |                                                             |
    |<--- 2. Response: 201 Created (request_uri, expires_in) -----|
    |                                                             |
    |---- 3. POST /oauth/token (with client cert & request_uri) ->|
    |                                                             |
    |                                                             |-- [Verify Binding]
    |                                                             |-- [Issue Token]
    |                                                             |
    |<--- 4. Response: 200 OK (access_token, dpop_signing_key) ---|
```

### 1.3.2 PAR Request Payload Example
```http
POST /oauth/par HTTP/1.1
Host: gateway.aethel.gov
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

response_type=code
&client_id=node_us_east_091
&scope=sovereign:settlement:write
&code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
&code_challenge_method=S256
```

### 1.3.3 PAR Response Payload Example
```json
{
  "request_uri": "urn:ietf:params:oauth:request_uri:b9c8d7e6f5a4b3c2d1e0f9a8b7c6d5e4",
  "expires_in": 60
}
```

---

## 1.4 Federal Reserve Payment Account Integration

Under the **May 19, 2026 Executive Order**, the Federal Reserve provides direct API access to sovereign Payment Accounts. The Aethel Sovereign Gateway acts as the translation layer between the high-speed M2M network and the Federal Reserve's internal ledger.

### 1.4.1 API Endpoint Specification
*   **Endpoint:** `POST /v1/settlement/transfer`
*   **Authentication:** OAuth 2.0 Access Token (bound to client certificate via `x5t#S256`) + DPoP (Demonstrating Proof-of-Possession) token.

### 1.4.2 Request Payload Schema (JSON)
```json
{
  "transaction_id": "tx_889201_aethel_001",
  "timestamp": "2026-05-20T04:00:00.000Z",
  "source_payment_account": "US-FED-PAY-0019283",
  "destination_payment_account": "US-FED-PAY-0099112",
  "amount": {
    "currency": "USD",
    "value": "150000000.00"
  },
  "settlement_type": "RTGS_IMMEDIATE",
  "routing_directives": {
    "bypass_ach": true,
    "force_fednow": true
  },
  "cryptographic_proof": {
    "signature": "MEQCID3y8X9...[truncated]...18a9f=",
    "algorithm": "Ed25519",
    "public_key_fingerprint": "sha256:9f8e7d6c5b4a3"
  }
}
```

---

## 1.5 Gateway Deployment & Bootstrap Protocol

To deploy the Aethel Sovereign Gateway as a native financial runtime environment, execute the following bootstrap sequence on a hardened, minimal Linux distribution (e.g., AethelOS / SELinux Enforced).

### 1.5.1 Step-by-Step Deployment Commands

1.  **Initialize Cryptographic Key Storage (HSM Integration):**
    Ensure the PKCS#11 module is loaded and the hardware security module is initialized with the Federal Root Authority's intermediate keys.
    ```bash
    pkcs11-tool --module /usr/lib/libsofthsm2.so --init-token --slot 0 --label "AethelHSM" --pin 20260519
    ```

2.  **Generate Gateway Identity Keypair inside the HSM:**
    ```bash
    pkcs11-tool --module /usr/lib/libsofthsm2.so --login --pin 20260519 \
      --keypairgen --key-type EC:prime256v1 --label "aethel-gateway-key" --id 01
    ```

3.  **Deploy the Gateway Runtime Binary:**
    The gateway runtime is compiled from the native Rust-based `aethel-core` engine.
    ```bash
    systemctl stop legacy-networking-services || true
    cp /opt/aethel/bin/aethel-gateway /usr/local/bin/
    chmod 700 /usr/local/bin/aethel-gateway
    ```

4.  **Apply Systemd Service Configuration:**
    Create `/etc/systemd/system/aethel-gateway.service`:
    ```ini
    [Unit]
    Description=Aethel Sovereign Gateway Runtime
    After=network.target
    ConditionPathExists=/etc/aethel/certs/federal_root_ca.crt

    [Service]
    Type=simple
    User=aethel-runtime
    Group=aethel-runtime
    ExecStart=/usr/local/bin/aethel-gateway --config /etc/aethel/gateway.conf
    Restart=always
    RestartSec=1
    LimitNOFILE=65536
    CapabilityBoundingSet=CAP_NET_BIND_SERVICE
    AmbientCapabilities=CAP_NET_BIND_SERVICE
    NoNewPrivileges=true
    ProtectSystem=strict
    ProtectHome=true
    ReadWritePaths=/var/log/aethel/

    [Install]
    WantedBy=multi-user.target
    ```

5.  **Enable and Start the Gateway:**
    ```bash
    systemctl daemon-reload
    systemctl enable aethel-gateway
    systemctl start aethel-gateway
    ```

---

## 1.6 Verification and Health Attestation

Once deployed, the gateway must perform a self-attestation routine before accepting external connections. It queries the local TPM 2.0 module to verify system integrity (PCR measurements) and submits the signed attestation document to the Federal Root Authority.

### 1.6.1 Attestation Verification Command
```bash
aethel-gateway --verify-attestation --tpm-path /dev/tpmrm0
```

Expected output:
```
[INFO] Initializing TPM 2.0 Attestation Verification...
[INFO] PCR 00: 9f83c12b... Verified.
[INFO] PCR 04: a1b2c3d4... Verified.
[INFO] Cryptographic handshake engine: mTLS 1.3 ACTIVE.
[INFO] Pushed Authorization Requests (PAR) endpoint: ONLINE.
[INFO] Federal Reserve Payment Account Interface: CONNECTED.
[SUCCESS] Aethel Sovereign Gateway is fully integrated and locked.
```

The gateway is now active. The legacy financial rails are bypassed. The sovereign state engine has established its perimeter.