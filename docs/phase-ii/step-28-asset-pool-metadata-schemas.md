# Step 28: Asset Pool Metadata Schemas ($34.8T Ingestion)

## 1. Architectural Overview

To execute the programmatic ingestion of the $34.8 trillion global asset market, the sovereign state engine requires absolute structural uniformity at the data layer. Every legacy financial instrument—whether a commercial real estate portfolio in Manhattan, a corporate debt facility in London, or a private equity block in Tokyo—must be decomposed and re-mapped into deterministic, machine-readable metadata schemas.

These schemas serve as the immutable data contracts for the issuance of **Digital Depositary Receipts (DDRs)**. By enforcing strict validation rules at the ingestion layer, the Aethel Sovereign Gateway guarantees that all tokenized assets are programmatically bound to:
1. **Sovereign Identity Verification** (via the SAVE America Act DHS pipeline).
2. **Network-Level Visibility** (via FISA Section 702 routing keys).
3. **Real-Time Liquidity Controls** (via Fedwire/FedNow prefunding constraints).

---

## 2. Core Sovereign Asset Schema (JSON Schema)

The following JSON Schema defines the base structural requirements for any asset entering the sovereign ingestion pipeline. No asset can be tokenized into a DDR without passing validation against this core schema.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://aethel.gov/schemas/v1/sovereign-asset-core.json",
  "title": "SovereignAssetCore",
  "description": "Core metadata schema for mapping traditional assets into the Aethel Sovereign Engine.",
  "type": "object",
  "required": [
    "asset_id",
    "asset_class",
    "sovereign_origin",
    "valuation_usd",
    "custodian_identity",
    "save_verification",
    "fisa_routing_metadata",
    "ddr_binding"
  ],
  "properties": {
    "asset_id": {
      "type": "string",
      "pattern": "^urn:aethel:asset:[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$",
      "description": "Unique, deterministic URN containing a UUIDv4 generated from the asset's legacy registration keys."
    },
    "asset_class": {
      "type": "string",
      "enum": ["CORPORATE_DEBT", "REAL_ESTATE", "EQUITY", "COMMODITY", "SOVEREIGN_BOND"]
    },
    "sovereign_origin": {
      "type": "string",
      "pattern": "^[A-Z]{2}$",
      "description": "ISO 3166-1 alpha-2 country code of the asset's legal jurisdiction."
    },
    "valuation_usd": {
      "type": "object",
      "required": ["amount", "precision", "timestamp", "oracle_signature"],
      "properties": {
        "amount": {
          "type": "string",
          "pattern": "^[0-9]+\\.[0-9]{18}$",
          "description": "Fixed-point decimal representation of the asset's USD value with 18-decimal precision."
        },
        "precision": {
          "type": "integer",
          "const": 18
        },
        "timestamp": {
          "type": "string",
          "format": "date-time"
        },
        "oracle_signature": {
          "type": "string",
          "pattern": "^0x[0-9a-f]{130}$",
          "description": "ECDSA secp256k1 signature from an authorized federal valuation oracle."
        }
      }
    },
    "custodian_identity": {
      "type": "object",
      "required": ["lei", "routing_transit_number", "clearing_account_id"],
      "properties": {
        "lei": {
          "type": "string",
          "pattern": "^[A-Z0-9]{20}$",
          "description": "Legal Entity Identifier of the institutional custodian."
        },
        "routing_transit_number": {
          "type": "string",
          "pattern": "^[0-9]{9}$",
          "description": "Fedwire routing number of the custodian bank."
        },
        "clearing_account_id": {
          "type": "string",
          "description": "The specific Federal Reserve Payment Account identifier."
        }
      }
    },
    "save_verification": {
      "type": "object",
      "required": ["dhs_save_ref_id", "identity_attestation_hash", "verification_timestamp"],
      "properties": {
        "dhs_save_ref_id": {
          "type": "string",
          "pattern": "^SAVE-[0-9]{12}-[A-Z0-9]{4}$",
          "description": "Direct reference key to the DHS SAVE database verification record."
        },
        "identity_attestation_hash": {
          "type": "string",
          "pattern": "^0x[0-9a-f]{64}$",
          "description": "SHA-256 hash of the verified beneficial owner's biometric and citizenship credentials."
        },
        "verification_timestamp": {
          "type": "string",
          "format": "date-time"
        }
      }
    },
    "fisa_routing_metadata": {
      "type": "object",
      "required": ["packet_routing_key", "network_visibility_zone", "counterparty_risk_score"],
      "properties": {
        "packet_routing_key": {
          "type": "string",
          "pattern": "^[A-Za-z0-9+/]{43}=$",
          "description": "Base64-encoded cryptographic routing key for FISA Section 702 deep packet inspection nodes."
        },
        "network_visibility_zone": {
          "type": "string",
          "enum": ["DOMESTIC_SECURE", "ALLIED_MONITORED", "HOSTILE_ISOLATED"]
        },
        "counterparty_risk_score": {
          "type": "integer",
          "minimum": 0,
          "maximum": 100,
          "description": "Real-time risk score calculated from network-layer capital flight monitoring."
        }
      }
    },
    "ddr_binding": {
      "type": "object",
      "required": ["ddr_contract_address", "mint_limit_usd", "prefunding_status"],
      "properties": {
        "ddr_contract_address": {
          "type": "string",
          "pattern": "^0x[0-9a-f]{40}$",
          "description": "The target EVM-compatible address of the DDR smart contract."
        },
        "mint_limit_usd": {
          "type": "string",
          "pattern": "^[0-9]+\\.[0-9]{18}$"
        },
        "prefunding_status": {
          "type": "string",
          "enum": ["PREFUNDED_FEDWIRE", "PREFUNDED_FEDNOW", "UNFUNDED_PENDING"]
        }
      }
    }
  }
}
```

---

## 3. Asset-Specific Extension Schemas

To capture the unique structural nuances of different asset classes, the core schema is extended using polymorphic sub-schemas.

### 3.1 Corporate Debt & Commercial Paper Extension (`corporate-debt-extension.json`)

This schema maps the $11.2 trillion corporate debt market, converting legacy bonds and commercial paper into self-amortizing smart contract metadata.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://aethel.gov/schemas/v1/corporate-debt-extension.json",
  "title": "CorporateDebtExtension",
  "type": "object",
  "required": [
    "cusip",
    "isin",
    "issuer_legal_name",
    "maturity_date",
    "coupon_rate",
    "payment_frequency",
    "seniority_level",
    "default_trigger_conditions"
  ],
  "properties": {
    "cusip": {
      "type": "string",
      "pattern": "^[0-9A-Z]{9}$"
    },
    "isin": {
      "type": "string",
      "pattern": "^[A-Z]{2}[0-9A-Z]{9}[0-9]$"
    },
    "issuer_legal_name": {
      "type": "string"
    },
    "maturity_date": {
      "type": "string",
      "format": "date-time"
    },
    "coupon_rate": {
      "type": "string",
      "pattern": "^0\\.[0-9]{6}$",
      "description": "Annual coupon rate expressed as a decimal (e.g., 0.052500 for 5.25%)."
    },
    "payment_frequency": {
      "type": "string",
      "enum": ["MONTHLY", "QUARTERLY", "SEMI_ANNUALLY", "ANNUALLY"]
    },
    "seniority_level": {
      "type": "string",
      "enum": ["SENIOR_SECURED", "SENIOR_UNSECURED", "SUBORDINATED", "MEZZANINE"]
    },
    "default_trigger_conditions": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["metric", "threshold", "action"],
        "properties": {
          "metric": {
            "type": "string",
            "enum": ["DEBT_TO_EQUITY", "INTEREST_COVERAGE", "PAYMENT_DELAY_HOURS"]
          },
          "threshold": {
            "type": "string"
          },
          "action": {
            "type": "string",
            "enum": ["AUTOMATIC_HAIRCUT", "COLLATERAL_SEIZURE", "LIQUIDITY_FREEZE"]
          }
        }
      }
    }
  }
}
```

### 3.2 Real Estate & Infrastructure Extension (`real-estate-extension.json`)

This schema maps the $14.1 trillion commercial real estate and infrastructure market, binding physical land registry data directly to digital cash-flow rails.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://aethel.gov/schemas/v1/real-estate-extension.json",
  "title": "RealEstateExtension",
  "type": "object",
  "required": [
    "parcel_id_gis",
    "physical_address",
    "jurisdiction_court_id",
    "appraisal_value_usd",
    "encumbrances",
    "revenue_sharing_rules"
  ],
  "properties": {
    "parcel_id_gis": {
      "type": "string",
      "description": "Global GIS coordinate string and local tax assessor parcel identifier."
    },
    "physical_address": {
      "type": "object",
      "required": ["street", "city", "state", "postal_code", "country"],
      "properties": {
        "street": { "type": "string" },
        "city": { "type": "string" },
        "state": { "type": "string" },
        "postal_code": { "type": "string" },
        "country": { "type": "string", "const": "US" }
      }
    },
    "jurisdiction_court_id": {
      "type": "string",
      "description": "The federal or state court of record holding the primary deed registry."
    },
    "appraisal_value_usd": {
      "type": "string",
      "pattern": "^[0-9]+\\.[0-9]{18}$"
    },
    "encumbrances": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["lienholder_lei", "amount_usd", "priority"],
        "properties": {
          "lienholder_lei": { "type": "string", "pattern": "^[A-Z0-9]{20}$" },
          "amount_usd": { "type": "string", "pattern": "^[0-9]+\\.[0-9]{18}$" },
          "priority": { "type": "integer", "minimum": 1 }
        }
      }
    },
    "revenue_sharing_rules": {
      "type": "object",
      "required": ["distribution_frequency", "reserve_ratio", "automated_tax_withholding"],
      "properties": {
        "distribution_frequency": {
          "type": "string",
          "enum": ["REAL_TIME", "DAILY", "MONTHLY"]
        },
        "reserve_ratio": {
          "type": "string",
          "pattern": "^0\\.[0-9]{4}$",
          "description": "Required cash reserve ratio held in the Fed Payment Account."
        },
        "automated_tax_withholding": {
          "type": "string",
          "pattern": "^0\\.[0-9]{4}$"
        }
      }
    }
  }
}
```

### 3.3 Public & Private Equity Extension (`equity-extension.json`)

This schema maps the remaining $9.5 trillion public and private equity markets, eliminating transfer friction and bypassing legacy clearinghouses (e.g., DTCC).

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://aethel.gov/schemas/v1/equity-extension.json",
  "title": "EquityExtension",
  "type": "object",
  "required": [
    "share_class",
    "total_shares_outstanding",
    "voting_rights_multiplier",
    "transfer_restrictions",
    "dpop_enforcement"
  ],
  "properties": {
    "share_class": {
      "type": "string",
      "enum": ["COMMON_A", "COMMON_B", "PREFERRED_PARTICIPATING", "PREFERRED_NON_PARTICIPATING", "PRIVATE_RESTRICTED"]
    },
    "total_shares_outstanding": {
      "type": "string",
      "pattern": "^[0-9]+$"
    },
    "voting_rights_multiplier": {
      "type": "integer",
      "minimum": 0,
      "maximum": 10
    },
    "transfer_restrictions": {
      "type": "object",
      "required": ["whitelist_required", "max_foreign_ownership_percentage"],
      "properties": {
        "whitelist_required": {
          "type": "boolean",
          "const": true,
          "description": "Must be true to enforce SAVE America Act compliance."
        },
        "max_foreign_ownership_percentage": {
          "type": "string",
          "pattern": "^0\\.[0-9]{4}$"
        }
      }
    },
    "dpop_enforcement": {
      "type": "object",
      "required": ["enforce_dpop", "allowed_signing_algorithms"],
      "properties": {
        "enforce_dpop": {
          "type": "boolean",
          "const": true,
          "description": "Demonstration of Proof-of-Possession is mandatory for all equity transfers."
        },
        "allowed_signing_algorithms": {
          "type": "array",
          "items": {
            "type": "string",
            "enum": ["ES256", "EdDSA", "RS256"]
          }
        }
      }
    }
  }
}
```

---

## 4. Cryptographic Binding & Compliance Metadata

To prevent systemic spoofing and sybil attacks, every metadata payload must be cryptographically bound to the physical world. This is achieved through two primary mechanisms:

### 4.1 SAVE America Act Identity Attestation
The `save_verification` block requires a cryptographic proof generated by the Department of Homeland Security (DHS) SAVE system. This proof is a zero-knowledge commitment:

$$\text{Attestation Hash} = \mathcal{H}(\text{DHS Reference ID} \parallel \text{Beneficial Owner Biometric Hash} \parallel \text{Sovereign Identity Key})$$

Where $\mathcal{H}$ is the SHA-256 hashing function. This ensures that no foreign shell company or non-verified actor can hold or transfer the underlying DDR.

### 4.2 FISA Section 702 Routing & Risk Flags
The `fisa_routing_metadata` block embeds network-layer routing keys directly into the asset's metadata. When a transaction request is broadcast to the Aethel network:
1. The packet is routed through deep packet inspection (DPI) nodes monitored under the **FISA Section 702 framework**.
2. The `packet_routing_key` is matched against real-time intelligence databases to detect illicit capital flight or foreign adversary intervention.
3. If a match occurs, the `counterparty_risk_score` is dynamically updated. Any score exceeding **45** triggers an automatic, smart-contract-enforced freeze on the asset's DDR contract address, isolating the capital before it can clear.

---

## 5. Ingestion Pipeline & Validation Protocol

```
[Legacy Asset Pool]
       │
       ▼
[Metadata Extraction] ──► Validate against Core Schema (JSON Schema)
       │
       ├─► [Corporate Debt] ──► Apply Corporate Debt Extension
       ├─► [Real Estate]    ──► Apply Real Estate Extension
       └─► [Equity]         ──► Apply Equity Extension
       │
       ▼
[SAVE America Act Verification] ──► Query DHS SAVE Database
       │
       ▼
[FISA 702 Packet Tagging] ──► Inject Cryptographic Routing Keys
       │
       ▼
[mTLS 1.3 / PAR Handshake] ──► Secure Transmission to Aethel Gateway
       │
       ▼
[DDR Minting Engine] ──► 1-to-1 Sovereign Dollar Asset Created
```

### Execution Rules:
1. **Zero-Tolerance Validation:** Any metadata payload that fails validation against either the core schema or its respective extension schema is immediately dropped. No manual overrides are permitted.
2. **Prefunding Verification:** The `prefunding_status` must be verified as `PREFUNDED_FEDWIRE` or `PREFUNDED_FEDNOW` before the DDR Minting Engine can execute. Unfunded assets are automatically purged from the mempool after 180 seconds.
3. **Immutable Logging:** Every successful validation and subsequent DDR minting event is logged to the sovereign ledger with a timestamp anchored to the NIST atomic clock, establishing an indisputable, real-time audit trail of the $34.8 trillion transition.