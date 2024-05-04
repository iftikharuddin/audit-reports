# Beanstalk Part 1 - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Arbitrage potential: Minting fertilizer using collateral assets' oracle price at 100% value without fee](#H-01)
    - ### [H-02. `sunrise` can be front-run, leads to un-fair reward distribution](#H-02)
    - ### [H-03. `UnwrapAndSendETH` can be called by anyone, leads to front-running and loss of user funds](#H-03)
    - ### [H-04. Harcoded `minAmountOut` and no `deadline` is present in sow, can lead to loss of tokens and sandwich attacks](#H-04)
    - ### [H-05. `gm` function is not validating account address, can lead to rewards loss](#H-05)
- ## Medium Risk Findings
    - ### [M-01. Lack of final validation step in beans distribution to fertilizer](#M-01)
    - ### [M-02. `C.bean().transfer` is not checking the return value, transaction may fails silently](#M-02)
    - ### [M-03. Missing max deposits check in SiloFacet `deposit` function, leads to manipulation of whitelisting assets](#M-03)
    - ### [M-04. Race condition in `TokenFacet`](#M-04)
    - ### [M-05. `getWellPriceFromTwaReserves` is reading stale price, can lead to discrepancies between the actual & calculated rewards](#M-05)
    - ### [M-06. `transferDeposits` has no validation for `recipient` address, can lead to deposits loss](#M-06)
- ## Low Risk Findings
    - ### [L-01. LibEthUsdOracle returning wrong price on `minAnswer`, impacting fertilizer minting](#L-01)
    - ### [L-02. Unsafe ABI Encoding](#L-02)
    - ### [L-03. `unwrapAndSendETH` is missing validation of recipient, user fund can be lost](#L-03)
    - ### [L-04. `switchUnderlyingToken` is missing require check for balanceOfUnderlying equals to 0](#L-04)
    - ### [L-05. `convertKind` is not checking if the return value `kind` is valid or not](#L-05)
    - ### [L-06. Length mismatch between stems and amounts can be passed to `enrootDeposits`,  can lead to run time errors or silent data corruption](#L-06)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Beanstalk

### Dates: Feb 26th, 2024 - Mar 25th, 2024

[See more contest details here](https://www.codehawks.com/contests/clsxlpte900074r5et7x6kh96)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 6
   - Low: Found 6 but valid and rewarded is one low.


# Low Risk Findings

## <a id='L-01'></a>L-01. LibEthUsdOracle returning wrong price on `minAnswer`, impacting fertilizer minting            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/libraries/Oracle/LibEthUsdOracle.sol#L63-L101

## Summary
The Chainlink aggregator utilized in the `LibEthUsdOracle` contract lacks a mechanism to detect and handle scenarios where the price of an asset falls outside of a predetermined price band. This limitation can result in the oracle returning the `minPrice` instead of the actual price of the asset during extreme market events, such as a significant drop in value. Consequently, users may continue to interact with the system, such as minting fertilizer tokens, using inaccurate price data. <a href="https://rekt.news/venus-blizz-rekt/" target="_blank"> similar case happened with Venus on BSC when LUNA imploded </a>

More Refs for similar issues like this:
- https://medium.com/cyfrin/chainlink-oracle-defi-attacks-93b6cb6541bf ( check Oracle Returns Incorrect Price During Flash Crashes )
- https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/18
- https://github.com/sherlock-audit/2023-05-ironbank-judging/issues/25

## Impact

The Chainlink aggregator can lead to potential exploitation of price discrepancies during extreme market conditions. For instance, if the price of an asset experiences a sudden crash, the oracle may continue to provide the `minPrice`, allowing users to conduct transactions at incorrect prices. This could result in financial losses for users and undermine the integrity of the system.

In our scenario, the `mintFertilizer` function within the FertilizerFacet contract, although it falls out of our immediate scope, relies on the `LibEthUsdOracle.getEthUsdPrice()` function (within our scope) to fetch the ETH/USD price from the Chainlink oracle. This price is crucial for calculating the amount of Fertilizer tokens that can be acquired with the provided `wethAmountIn` of WETH. However, if this function returns the `minPrice` during extreme market events, it would not reflect the actual price of the asset. Consequently, users could continue to mint fertilizer tokens using this **inaccurate price data**, leading to transactions occurring at incorrect prices.

## Recommendation
It is recommended to enhance the Chainlink oracle (`LibEthUsdOracle`) by implementing a mechanism to check the returned answer against predefined `minPrice` and `maxPrice` bounds. If the answer falls outside of these bounds, the oracle should revert the transaction, indicating that the price data is not reliable due to market conditions.


