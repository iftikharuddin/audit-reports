# AAVE Aptos V3 Security Findings

**Audit Contest:** [Cantina - AAVE Aptos](https://cantina.xyz/code/ad445d42-9d39-4bcf-becb-0c6c8689b767)  
**Auditor:** 0xTheBlackPanther  
**Date:** May 2025

---

## Finding #1: Address mismatch in configuration data storage & retrieval

| Severity | Status | Link |
|----------|--------|------|
| **High** | Confirmed | [Finding #19](https://cantina.xyz/code/ad445d42-9d39-4bcf-becb-0c6c8689b767/findings/19) |

### Summary
Critical address mismatch where configuration data is stored at `@aave_data` but all getter functions attempt to read from `@aave_pool`, causing complete protocol failure.

### Vulnerability Details

**Storage (aave-data/sources/v1.move):**
```move
fun init_module(account: &signer) {
    assert!(signer::address_of(account) == @aave_data, ...);
    move_to(account, Data { ... }); // Stores at @aave_data
}
```

**Retrieval (all getters read from wrong address):**
```move
public inline fun get_price_feeds_tesnet(): ... {
    &borrow_global<Data>(@aave_pool).price_feeds_testnet // ❌ Wrong address
}

public inline fun get_underlying_assets_testnet(): ... {
    &borrow_global<Data>(@aave_pool).underlying_assets_testnet // ❌ Wrong address
}

public inline fun get_reserves_config_testnet(): ... {
    &borrow_global<Data>(@aave_pool).reserves_config_testnet // ❌ Wrong address
}
// ... same pattern for all getters
```

### Impact
- Protocol completely non-functional at launch
- All transactions abort with `ERESOURCE_DNE` (Resource does not exist)
- No deposits, loans, or liquidations possible

### Proof of Concept
```move
#[test_only]
module aave_data::simple_verification {
    #[test]
    fun verify_bug_exists() {
        assert!(@aave_data != @aave_pool, 999);
        // If passes: Data stored at @aave_data, read from @aave_pool = BUG
    }
}
```

---

## Finding #2: Missing Chainlink Oracle Staleness Check

| Severity | Status | Link |
|----------|--------|------|
| **Medium** | Confirmed | [Finding #80](https://cantina.xyz/code/ad445d42-9d39-4bcf-becb-0c6c8689b767/findings/80) |

### Summary
Oracle module fetches price from Chainlink via `chainlink::get_benchmark_value(benchmark)` but doesn't validate the timestamp, allowing stale prices to be used.

### Vulnerability Details
Per [Chainlink Aptos Documentation](https://docs.chain.link/data-feeds/aptos), the benchmark returns both price and timestamp. The implementation:
- ✅ Fetches price
- ❌ Ignores timestamp
- ❌ No staleness validation

### Impact
- Stale oracle prices used for critical operations
- Incorrect collateral valuations
- Users can borrow/liquidate at manipulated prices
- Protocol insolvency risk during Chainlink downtime

### Recommendation
```move
let (price, timestamp) = chainlink::get_benchmark_value(benchmark);
assert!(timestamp::now_seconds() - timestamp < MAX_STALENESS_THRESHOLD, E_STALE_PRICE);
```

---

## Summary

| # | Title | Severity |
|---|-------|----------|
| 1 | Address Mismatch in Configuration Data Storage & Retrieval | High |
| 2 | Missing Chainlink Oracle Staleness Check | Medium |