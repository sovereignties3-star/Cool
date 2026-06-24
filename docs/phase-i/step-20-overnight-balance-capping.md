# Protocol: Operational Alpha // Phase I // Step 20
## Module: Overnight Balance Capping (OBC)
### Target: Institutional Node Liquidity Neutralization

```rust
// Aethel Sovereign Gateway // Runtime Environment
// Module: OBC-20-RTGS-Prefunding
// Status: Hard-Enforced // Immutable

use crate::sovereign::ledger::{NodeID, Balance, Currency};
use crate::fisa::network_visibility::FISA_702_STREAM;
use crate::error::SovereignError;

const MAX_OVERNIGHT_LIMIT: u128 = 1_000_000_000_000_000; // $1B in micro-units

/// Enforces the $1B overnight closing balance cap per institutional node.
/// This function executes at the T-minus 00:00:01 UTC daily settlement window.
/// Any excess liquidity is automatically swept into the Sovereign Consolidation Fund.
pub fn enforce_overnight_balance_cap(node_id: NodeID) -> Result<(), SovereignError> {
    let current_balance = fetch_node_balance(node_id)?;

    if current_balance > MAX_OVERNIGHT_LIMIT {
        let excess = current_balance - MAX_OVERNIGHT_LIMIT;
        
        // Log event via FISA-702 metadata stream for risk-mitigation profiling
        FISA_702_STREAM.log_event(
            node_id, 
            "EXCESS_LIQUIDITY_SWEEP", 
            excess
        );

        // Execute programmatic sweep to neutralize shadow banking leverage
        execute_sovereign_sweep(node_id, excess)?;
        
        // Update node state to reflect hard-capped status
        update_node_ledger(node_id, MAX_OVERNIGHT_LIMIT)?;
    }

    Ok(())
}

fn execute_sovereign_sweep(node: NodeID, amount: u128) -> Result<(), SovereignError> {
    // Atomic transfer to the Federal Reserve Payment Account framework
    // Bypassing legacy ACH/Correspondent rails
    ledger::transfer(
        node,
        "FED_SOVEREIGN_CONSOLIDATION_POOL",
        amount,
        Currency::USD_DDR
    )
}
```

---
**System Note:** The $1 billion limit is non-negotiable. Any attempt to circumvent this via synthetic derivative layering will trigger an immediate node-level audit via the FISA Section 702 visibility layer. The architecture is absolute.