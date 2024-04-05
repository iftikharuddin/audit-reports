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
   - Low: 6


# High Risk Findings

## <a id='H-01'></a>H-01. Arbitrage potential: Minting fertilizer using collateral assets' oracle price at 100% value without fee            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/beanstalk/barn/FertilizerFacet.sol#L63-L93

## Summary

The current implementation allows users to mint Fertilizer tokens using collateral assets at 100% of their value based on Oracle prices without any fees. This exposes the system to risks of arbitrage exploitation and may lead to a decrease in the quality of collateral for Fertilizer.

**Note:** While the `mintFertilizer` function itself is out of scope, the Oracle it relies on, specifically `LibEthUsdOracle.getEthUsdPrice`, is within scope due to its potential to produce issues.


## Impact
The Oracle price can not be trusted as the real-time price.

For example, the `BTC/USD` and `ETH/USD` price feeds on miannet have a "Deviation threshold" of `0.5%`, meaning that the price will only be updated once the price movement exceeds 0.5% within the heartbeat period.

1. **Arbitrage Exploitation**: Without imposing a minting fee or considering the potential deviation in Oracle prices, users can exploit price differences between the collateral assets and their actual market value. This can lead to excessive minting of Fertilizer tokens without proper collateral backing, ultimately compromising the stability and integrity of the system.

2. **Quality of Collateral**: Continuous minting of Fertilizer tokens without considering the actual market value of collateral assets may result in a decrease in the quality of collateral backing the tokens. This could lead to a situation where the value of the collateral is insufficient to cover the value of the minted Fertilizer tokens, posing a significant risk to the overall stability of the system.

3. **Oracle Price Reliability**: The reliance on Oracle prices without considering their real-time accuracy or potential deviations introduces uncertainty into the minting process. Users may inadvertently rely on outdated or inaccurate price information, further exacerbating the risks associated with arbitrage and collateral quality.

## Recommendation

Consider adding a minting fee of `0.5%` to `1%` (it should be higher than the deviation)
## <a id='H-02'></a>H-02. `sunrise` can be front-run, leads to un-fair reward distribution            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/beanstalk/sun/SeasonFacet/SeasonFacet.sol#L30-L57

## Summary

The `SeasonFacet` function `sunrise` which advances Beanstalk to the next Season and rewards the caller with beans. There is a vulnerability present in this implementation that allows front-running of the reward mechanism. This vulnerability enables an attacker to consistently front-run legitimate callers and claim the rewards for themselves, leaving the legitimate callers unsatisfied.

## Impact

It undermines the fairness of the reward distribution system. It allows an attacker to **monopolize** the rewards meant for legitimate users, leading to dissatisfaction among users who may lose gas fees without receiving the expected rewards.

## Proof of concept

1. Bob calls the `sunrise` function to advance Beanstalk to the next Season and receive rewards.
2. Before Bob's transaction is processed, Alice front-runs Bob's transaction and calls the `sunrise` function herself.
3. Alice successfully advances Beanstalk to the next Season and receives the rewards meant for Bob.
4. Bob's transaction is reverted, causing him to lose gas fees and miss out on the rewards.

This scenario highlights how an attacker like Alice can exploit the front-running vulnerability to consistently claim rewards meant for other users, leading to unfairness and dissatisfaction among legitimate users.

## Recommendation

Implement a front-running mitigation mechanism to ensure fair rewards distribution and prevent unauthorized claim of rewards.
## <a id='H-03'></a>H-03. `UnwrapAndSendETH` can be called by anyone, leads to front-running and loss of user funds            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/pipeline/junctions/UnwrapAndSendETH.sol#L27-L35

## Summary

The `UnwrapAndSendETH` contract allows anyone to send WETH to it and subsequently withdraw and transfer the received funds to a specified address through the `unwrapAndSendETH` function. This design presents a vulnerability to frontrunning, enabling an attacker to preemptively withdraw and transfer funds sent by another user.

Even though `UnwrapAndSendETH` is a junction contract in relation to the pipeline contract. Junctions are helper contracts that can be used in a pipeline call to unlock greater functionality.  However there are no clear docs or instructions which stops the users from using this contract in isolation.

## Vulnerability Details

The contract utilizes the `receive() external payable {}` function to accept WETH from any address. The `unwrapAndSendETH` function then facilitates the withdrawal and transfer of WETH to a specified address without adequate validation of the caller's identity.

## Impact

This vulnerability allows an attacker to exploit the contract by front-running a legitimate user's attempt to withdraw and transfer funds. Consequently, the attacker gains control over the funds intended for the original sender.

## PoC

1. Bob calls `unwrapAndSendEth` & sends WETH to the contract.
2. Alice notices the WETH in the contract.
3. Alice front-runs and calls `unwrapAndSendEth` first, withdrawing and transferring all the funds to herself.
4. Bob loses the funds he sent to the contract.

This sequence of actions allows an attacker (in this case, Alice) to exploit the contract and transfer funds intended for another user (Bob) to their own address.

## Recommendations

Implement Access Control: Introduce access control mechanisms to restrict the ability to call the `unwrapAndSendETH` function. Only the original sender or authorized addresses should be allowed to withdraw and transfer funds.


## <a id='H-04'></a>H-04. Harcoded `minAmountOut` and no `deadline` is present in sow, can lead to loss of tokens and sandwich attacks            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L203-L210

## Summary
The `sop` function in the Beanstalk protocol contains a vulnerability where the `swapFrom` call has hardcoded values for `minAmountOut` (slippage protection) and `deadline`. The absence of slippage protection and the disabled deadline check makes the protocol susceptible to sandwich attacks, MEV exploits, and potential significant loss of tokens.

## Impact
The lack of slippage protection and the disabled deadline check expose users to the risk of receiving 0 output tokens and allow transactions to be executed at unfavorable times. This vulnerability can result in substantial financial losses for users.

## PoC:
- When `swapFrom` is called in here ( https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/main/protocol/contracts/beanstalk/sun/SeasonFacet/Weather.sol#L203-L210 ) the minAmountOut is hardcoded to `0` and the deadline check is set to `type(uint256).max`, which means the deadline check is disabled!
- When `sop()` function is called it will try to perform the swap, Then while the transaction is in the mempool, here "minTokensOut" is hard-coded to 0 so the swap can potentially return 0 output tokens, and the deadline parameter is hard-coded to the max value of `utint256`, so the transaction can be held & executed at a much later & more unfavorable time to the user. This combination of no Slippage & no Deadline exposes the user to the potential loss of all their input tokens! 

## Recommendation
Allow user to specify slippage parameters  `minAmountOut` and `deadline`
## <a id='H-05'></a>H-05. `gm` function is not validating account address, can lead to rewards loss            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/beanstalk/sun/SeasonFacet/SeasonFacet.sol#L44-L57

## Summary
 The `gm` function in the `SeasonFacet` advances Beanstalk to the next season, sending reward Beans to a specified address and balance. However, if the specified account address is entered as the zero address due to any reason or by mistake, it will lead to a loss of rewards, especially when the mode is set to `To.INTERNAL`.

## PoC Flow

1. The `gm` function is called with `account` set to the zero address and `mode` set to `To.INTERNAL`.
2. The `gm` function calls the `incentivize` function.
3. The `incentivize` function calls `LibTransfer.mintToken` with the zero address as the `account` parameter.
4. In the `sendToken` function, when `mode == To.INTERNAL`, it calls `LibBalance.increaseInternalBalance`.
5. The `increaseInternalBalance` function uses `getInternalBalance` and `setInternalBalance`.
6. If `getInternalBalance` is called with the zero address as `account`, it will fetch an incorrect balance.
7. When `setInternalBalance` is called with the zero address as `account`, it will set the new balance to the zero address, resulting in a loss of rewards.

## Impact

It can lead to the loss of rewards for the specified account when the zero address is unintentionally provided. This is particularly critical when the mode is set to `To.INTERNAL`, as it directly modifies the internal balance associated with the account.

## Recommendations

Implement input validation in the `gm` function to ensure that the `account` parameter is not the zero address. This can be achieved by adding a require statement at the beginning of the function.

   ```solidity
   require(account != address(0), "Invalid account address");
   ```

Another thing is if these functions are used widely and it will be because this is a library, so better add checks in the relevant functions (`getInternalBalance` and `setInternalBalance`) to ensure that the zero address is not used, preventing unexpected behavior.


		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Lack of final validation step in beans distribution to fertilizer            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/beanstalk/sun/SeasonFacet/Sun.sol#L106-L149

## Summary
The function `rewardToFertilizer` is responsible for distributing Beans to fertilizers. However, a critical issue has been identified where the distribution logic lacks a final validation step, potentially leading to discrepancies between the amount distributed and the amount owed.

## Impact
The issue arises when the distribution loop ends prematurely due to the absence of available fertilizers. In this scenario, the final distribution of Beans to the remaining fertilizers occurs outside the loop, where there is no validation to ensure that all owed Beans are indeed distributed. This oversight poses a risk of misallocation of Beans, which could result in inconsistencies between the distributed and owed amounts.

## Recommendation
To address this issue, it's recommended to implement a final validation step outside the loop to ensure that all owed Beans are distributed accurately. This can be achieved by placing a `require` statement after the loop to validate that the total amount distributed equals the total amount owed across all fertilizers.

For example new code will look like this:
```diff
     /**
     * @dev Distributes Beans to Fertilizer.
     */
    function rewardToFertilizer(uint256 amount) internal returns (uint256 newFertilized) {
        // 1/3 of new Beans being minted
        uint256 maxNewFertilized = amount.div(FERTILIZER_DENOMINATOR);

        // Get the new Beans per Fertilizer and the total new Beans per Fertilizer
        uint256 newBpf = maxNewFertilized.div(s.activeFertilizer); // @audit can s.activeFertilizer be zerO?
        //Store the current Beans per Fertilizer (s.bpf) in oldTotalBpf and
        uint256 oldTotalBpf = s.bpf;
        //calculate the new total Beans per Fertilizer by adding newBpf to oldTotalBpf, storing the result in newTotalBpf.
        uint256 newTotalBpf = oldTotalBpf.add(newBpf);

        // Get the end Beans per Fertilizer of the first Fertilizer to run out.
        uint256 firstEndBpf = s.fFirst; //3

        //Loop to Distribute Beans
        //While the new total Beans per Fertilizer (newTotalBpf) is greater than or equal to the Beans per Fertilizer
        //of the first Fertilizer to run out (firstEndBpf), execute the loop.
        //If the next fertilizer is going to run out, then step BPF according
        while (newTotalBpf >= firstEndBpf) { // 3
            // Calculate BPF and new Fertilized when the next Fertilizer ID ends
            newBpf = firstEndBpf.sub(oldTotalBpf);
            newFertilized = newFertilized.add(newBpf.mul(s.activeFertilizer));

            // If there is no more fertilizer, end
            if (!LibFertilizer.pop()) {
                s.bpf = uint128(firstEndBpf); // SafeCast unnecessary here.
                s.fertilizedIndex = s.fertilizedIndex.add(newFertilized);
                require(s.fertilizedIndex == s.unfertilizedIndex, "Paid != owed");
                return newFertilized;
            }

            // Calculate new Beans per Fertilizer values
            newBpf = maxNewFertilized.sub(newFertilized).div(s.activeFertilizer);
            oldTotalBpf = firstEndBpf;
            newTotalBpf = oldTotalBpf.add(newBpf); //4
            firstEndBpf = s.fFirst; // 3
        }

        // Distribute the rest of the Fertilized Beans
        s.bpf = uint128(newTotalBpf); // SafeCast unnecessary here.
        newFertilized = newFertilized.add(newBpf.mul(s.activeFertilizer));
        s.fertilizedIndex = s.fertilizedIndex.add(newFertilized);

+       // Recommended final validation step
+        require(s.fertilizedIndex == s.unfertilizedIndex, "Paid != owed");
}
```
## <a id='M-02'></a>M-02. `C.bean().transfer` is not checking the return value, transaction may fails silently            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/libraries/Convert/LibWellConvert.sol#L202

## Summary

The `LibWellConvert.sol` uses the `transfer` function to transfer the specified amount of Beans to the well address, which represents the liquidity pool. However, it fails to handle the boolean return value from these external calls, creating a potential risk of **silent transaction failures**.

There is no return value checked after the transfer, which could potentially lead to issues if the transfer fails or the expected amount of Beans is not transferred. It's important to handle potential failure scenarios appropriately, such as reverting the transaction or implementing error handling logic.

## Impact:

Failure to handle the return value could result in silent transaction failures. If the transfer operation fails for any reason, such as insufficient funds or other unforeseen issues, the contract will not be aware of the failure, potentially leading to unexpected behavior and user confusion.

## Recommendation:

Implement proper error handling logic after the `transfer` function calls to check for the boolean return value and handle potential failure scenarios accordingly. This may include reverting the transaction if the transfer fails or emitting an event to notify users about the failure.

For example:
```diff
/**
 * @dev Adds as Beans Liquidity with the constraint that delta B >= 0.
 */
function _wellAddLiquidityTowardsPeg(uint256 beans, uint256 minLP, address well)
    internal
    returns (uint256 lp, uint256 beansConverted)
{
    (uint256 maxBeans,) = _beansToPeg(well);
    require(maxBeans > 0, "Convert: P must be >= 1.");
    beansConverted = beans > maxBeans ? maxBeans : beans;

    // Transfer Beans to well address
+    bool success = C.bean().transfer(well, beansConverted);
+   require(success, "Convert: Bean transfer to well failed"); // Revert if transfer fails

    lp = IWell(well).sync(address(this), minLP);
}
```

## <a id='M-03'></a>M-03. Missing max deposits check in SiloFacet `deposit` function, leads to manipulation of whitelisting assets            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/beanstalk/silo/SiloFacet/SiloFacet.sol#L48-L66


## Summary

The deposit function within the `SiloFacet` has no checks for maximum deposit amounts. The potential risks associated with unlimited deposits, leading to excessive voting power and rewards share of Bean seignorage. Additionally, the lack of constraints on deposit amounts could enable users to abuse voting mechanisms to influence asset whitelisting decisions.

## Impact

1. **Unlimited Voting Power**: Unlimited deposits could result in users accumulating excessive voting power within the system, allowing them to exert disproportionate influence over governance decisions, such as asset whitelisting. This undermines the integrity and fairness of the governance process.

2. **Excessive Rewards Share**: Unlimited deposits may lead to users receiving disproportionately large rewards shares of Bean seignorage, potentially disrupting the economic equilibrium of the ecosystem and affecting the distribution of rewards among participants.

3. **Governance Manipulation**: Users with unlimited deposits could manipulate governance mechanisms to prioritize the whitelisting of specific assets that align with their interests, potentially undermining the diversity and inclusivity of the ecosystem.

## Recommendation
 Introduce checks to enforce minimum and maximum deposit amounts, ensuring that deposit sizes are within reasonable limits. Define sensible thresholds based on factors such as system capacity, economic considerations, and risk management.


## <a id='M-04'></a>M-04. Race condition in `TokenFacet`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/beanstalk/farm/TokenFacet.sol#L169-L184

## Summary

The `permitToken` function within the `TokenFacet`, has the potential race condition issues associated with the permit system. Even though `TokenFacet` is out of scope of this audit but the permit function allows users to create approvals prior to execution, which inherently introduces the risk of race conditions. And these approvals of tokens can be used in Beanstalk instances which are in scope. The implementation of the permit function does not include explicit safeguards against race conditions, leaving room for potential vulnerabilities.

## Impact

**Race Conditions**: The permit function's design exposes it to race conditions, where multiple transactions may attempt to modify allowances concurrently, leading to inconsistencies or unexpected behavior. This could potentially allow attackers to exploit the system by manipulating allowances or gaining unauthorized access.

## PoC

1. The `LibTokenPermit` permit function is called by the `TokenFacet` `permitToken` function.
2. After the `permit` function call, the `permitToken` function invokes the `approve` function from `LibTokenApprove`.
3. The `approve` function updates the approval for the specified spender and token to the provided amount. It updates the state variable `s.a[account].tokenAllowances[spender][token]` with the new allowance amount.

The `approve` function updates the allowance for the specified spender and token to the provided amount. Therefore, it will overwrite any previous approval for that spender and token with the new amount. 

There is a potential for a race condition if multiple transactions are attempting to update the allowance for the same spender and token simultaneously. If two transactions read the allowance, modify it, and then attempt to write it back without awareness of each other, it could result in inconsistencies or unexpected behavior.

## Recommendation

Permit systems in general have race conditions given the signature is created prior to execution. So generally it’s up to the implementor to properly implement the permit function such that a race condition doesn’t occur, this is why provide clear documentation and guidance to users regarding the risks associated with creating approvals and using the permit function. 
## <a id='M-05'></a>M-05. `getWellPriceFromTwaReserves` is reading stale price, can lead to discrepancies between the actual & calculated rewards            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/beanstalk/sun/SeasonFacet/SeasonFacet.sol#L100-L101

## Summary

The `SeasonFacet` calculates rewards for users calling the `sunrise` function using the stale price fetched from `LibWell.getWellPriceFromTwaReserves(C.BEAN_ETH_WELL)`. This method reads the `Bean/ETH` price calculated by the Minting Well from storage, resulting in the retrieval of stale price data. This stale price calculation can lead to discrepancies between the actual and calculated rewards.

## Impact

Using stale price data to calculate rewards introduces a significant risk of inaccurate reward distribution. If the live price of `Bean/ETH` deviates from the stale price used for reward calculation, users may receive more or fewer rewards than intended. This discrepancy can result in protocol losses or user dissatisfaction, undermining the fairness and integrity of the reward system.

## Recommendations

It is recommended to update the reward calculation mechanism in the `SeasonFacet` contract to fetch the live price of Bean/ETH from an oracle or another reliable source instead of relying on stale price data. By incorporating live price data, the reward distribution process can accurately reflect market conditions, ensuring fair and transparent rewards for users.
## <a id='M-06'></a>M-06. `transferDeposits` has no validation for `recipient` address, can lead to deposits loss            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/beanstalk/silo/SiloFacet/SiloFacet.sol#L174-L196

### Summary:

The `transferDeposits` function in the provided `SiloFacet` facilitates the transfer of multiple deposits between addresses. However, it lacks a validation check for the recipient address, allowing the possibility of transferring deposits to the zero address. This oversight could lead to the loss of assets for the sender. 

## Impact:

The absence of a recipient address validation check in the `transferDeposits` function poses a critical risk. If a user unintentionally or maliciously provides the zero address as the recipient, all deposits from the sender's account may be lost, leading to irreversible consequences. This could result in financial losses for users and undermine the security and reliability of the protocol. 

## Recommendations:

**Recipient Address Validation:**
   Implement a check within the `transferDeposits` function to ensure that the recipient address is not the zero address. This will prevent the accidental or intentional transfer of deposits to an invalid or unrecoverable destination.

```
require(recipient != address(0), "Silo: Invalid recipient address (zero address)");
```



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


## <a id='L-02'></a>L-02. Unsafe ABI Encoding            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/libraries/Silo/LibWhitelist.sol#L236

## Summary
`abi.encodeWithSelector` is used in below files generate calldata for a low-level call.  `abi.encodeWithSelector` is not type-safe and this method is error-prone and should be considered unsafe.

- LibWhitelist function `verifyGaugePointSelector`
- LibEvaluate function `getLiquidityWeight`
- LibTokenSilo function `encodeBdvFunction`
- and LibGauge function `calcGaugePoints`


## Recommendations

Consider replacing all instances of unsafe ABI encoding with `abi.encodeCall` . It checks whether the supplied values actually match the types expected by the called function and also avoids errors caused by typos.
## <a id='L-03'></a>L-03. `unwrapAndSendETH` is missing validation of recipient, user fund can be lost            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/pipeline/junctions/UnwrapAndSendETH.sol#L27-L35

## Summary

The issue arises from the possibility of users mistakenly entering the zero address (`0x000...`) as the recipient (`to`) address when calling the `unwrapAndSendETH` function in the `UnwrapAndSendETH` contract. This oversight can result in the loss of funds as the ETH transferred to the zero address cannot be recovered.

## Impact

If a user mistakenly provides the zero address as the recipient when calling the `unwrapAndSendETH` function, the ETH transferred from the contract will be irreversibly lost. This can lead to financial losses for the user and may impact the usability and trustworthiness of the contract.

## Recommendation

To mitigate this issue, the following steps can be taken:

**Input Validation**: Implement input validation to ensure that the `to` address provided is not the zero address and is a valid Ethereum address format.

E.g new code will look like this:

```solidity
/// @notice Unwrap WETH and send ETH to the specified address
/// @dev Make sure to load WETH into this contract before calling this function
function unwrapAndSendETH(address to) external {
    // Ensure the 'to' address is not zero
    require(to != address(0), "Invalid 'to' address");

    // Validate the 'to' address format
    require(to != address(this), "Invalid 'to' address"); // Prevent sending to this contract itself

    // Check WETH balance
    uint256 wethBalance = IWETH(WETH).balanceOf(address(this));
    require(wethBalance > 0, "Insufficient WETH");

    // Withdraw WETH and transfer ETH to the specified address
    IWETH(WETH).withdraw(wethBalance);
    (bool success,) = to.call{value: address(this).balance}(new bytes(0));
    require(success, "ETH transfer failed. Check 'to' address.");
}
```
## <a id='L-04'></a>L-04. `switchUnderlyingToken` is missing require check for balanceOfUnderlying equals to 0            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/libraries/LibUnripe.sol#L134-L141

## Summary
The `switchUnderlyingToken` function within the `LibUnripe` library lacks a required check to ensure that `s.u[unripeToken].balanceOfUnderlying` is zero before allowing the underlying token to be switched. This omission could potentially lead to misuse of the function and violate the main invariant.

## Impact
The absence of this check increases the risk of unintended behavior and could result in inconsistencies within the application's state. Developers or users may inadvertently call the function without ensuring that the balance of the underlying token is zero, which can lead to unexpected outcomes and compromise the integrity of the system.

## PoC

The `InitMigrateUnripeBean3CrvToBeanEth.sol`[https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/beanstalk/init/InitMigrateUnripeBean3CrvToBeanEth.sol#L22-L33] file calls `LibUnripe.switchUnderlyingToken(C.UNRIPE_LP, C.BEAN_ETH_WELL);` without the required check, despite being out of scope, potentially leading to unintended consequences.

## Recommendations
It is recommended to add a `require` statement within the `switchUnderlyingToken` function to enforce the condition that `s.u[unripeToken].balanceOfUnderlying` must be zero before proceeding with the switch.

E.g new code should look like this:

```diff
/**
 * @dev Switches the underlying token of an unripe token.
 * Should only be called if `s.u[unripeToken].balanceOfUnderlying == 0`.
 */
function switchUnderlyingToken(address unripeToken, address newUnderlyingToken) internal {
    AppStorage storage s = LibAppStorage.diamondStorage();
+  require(s.u[unripeToken].balanceOfUnderlying == 0, "Unripe: Underlying balance > 0");
    s.u[unripeToken].underlyingToken = newUnderlyingToken;
    emit SwitchUnderlyingToken(unripeToken, newUnderlyingToken);
}
```
## <a id='L-05'></a>L-05. `convertKind` is not checking if the return value `kind` is valid or not            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/libraries/Convert/LibConvertData.sol#L23-L30

## Summary
In the `convertKind` function, consider adding a default case to handle potential errors or unexpected values. This helps ensure that the function can gracefully handle future modifications to the enum. or even if someone passed a wrong kind so it will return a graceful error rather then returning something weird.

## Recommendations

Add this check

```diff
/// @notice Decoder for the Convert Enum
function convertKind(bytes memory self) internal pure returns (ConvertKind) {
    ConvertKind kind = abi.decode(self, (ConvertKind));
+   require(kind <= ConvertKind.UNRIPE_TO_RIPE, "Invalid ConvertKind");
 +  return kind;
}
```
## <a id='L-06'></a>L-06. Length mismatch between stems and amounts can be passed to `enrootDeposits`,  can lead to run time errors or silent data corruption            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/beanstalk/silo/EnrootFacet.sol#L133-L198


## Summary

The `EnrootFacet` exhibits a potential issue related to a length mismatch between the `stems` and `amounts` arrays in the `enrootDeposits` and `removeDepositsFromAccount` functions. The absence of a length check may lead to unexpected behavior or runtime errors.

## Impact

1. **Runtime Errors**: The absence of a length check in the provided code may lead to runtime errors during the execution of the `enrootDeposits` and `removeDepositsFromAccount` functions if the lengths of the `stems` and `amounts` arrays are not the same.

2. **Unexpected Behavior**: In scenarios where the lengths of `stems` and `amounts` arrays differ, the contract might not execute the intended logic correctly. 

3. **User experience**: User will get frustrated if he is not getting a proper error message on revert.

## PoC
- Assume the `unripeBean` is whitelisted and unripe token
- Copy the below test and run `forge test --match-test testEnrootDeposits -vvvv` cmd

```solidity
function testEnrootDeposits() public {
    // Example stems and amounts arrays with the diff length
    int96[] memory stems = new int96[](3);
    stems[0] = 100;
    stems[1] = 200;
    stems[2] = 300;

    uint256[] memory amounts = new uint256[](1);
    amounts[0] = 1000;

    enrootFacet.enrootDeposits(address (unripeBean), stems, amounts);
}
```

Result if `amount < steps`:

```
    ├─ [47808] EnrootFacet::enrootDeposits(UnripeBean: [0xDB8979F5f6e3c44F8312097E3E424543d0156cD8], [100, 200, 300], [1000])
    │   └─ ← EvmError: **InvalidFEOpcode**
    └─ ← EvmError: Revert
```

Result if `amount > steps`:
```
    │   ├─ [102] EnrootFacet::00000000(00000000000000000000000000000000000000000000000000000000000003e8) [staticcall]
    │   │   └─ ← ()
    │   └─ ← EvmError: Revert
    └─ ← EvmError: Revert
```

## Recommendations

Add a quick check at the start of the `enrootDeposits` function to make sure the lengths of the `stems` and `amounts` lists are the same.

```solidity
// Check if the array lengths match
require(stems.length == amounts.length, "Silo: Length mismatch between stems and amounts");
```

After investigating further, it turns out this function is also used in `_withdrawDeposits`, which already has a similar check. So, it makes sense to suggest adding this length check directly to the shared `LibSilo` library. This way, every time `_removeDepositsFromAccount` is called, it will automatically handle the check, making the code cleaner and avoiding repetition.


