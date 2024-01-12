# stake.link - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Unauthorized withdrawals without unlock initiation check in `_executeQueuedLockUpdates`](#H-01)
    - ### [H-02. Potential Division by zero in `distributeRewards` function](#H-02)
    - ### [H-03. Array length mismatch in `_buildCCIPMessage` function, may lead to index out of bounds errors](#H-03)
    - ### [H-04. Potential Reentrancy Vulnerability in `handleIncomingRESDL` function](#H-04)
    - ### [H-05. Mismatched Arrays in `distributeRewards` function](#H-05)
- ## Medium Risk Findings
    - ### [M-01. Unchecked same-chain transfer vulnerability, user can lose tokens](#M-01)
    - ### [M-02. Incomplete token approval management in `handleOutgoingRESDL` function](#M-02)
    - ### [M-03. Array length mismatch vulnerability in `performUpkeep`](#M-03)
    - ### [M-04. Potential dust accumulation issue in Reward Distribution function](#M-04)
    - ### [M-05. Missing check for maximum locking duration in migrate function](#M-05)
    - ### [M-06. Potential risks in `RESDLTokenBridge` extraArgs configuration](#M-06)
    - ### [M-07. Potential minting disruption and duplicate entry addition in `SDLPoolSecondary` Contract](#M-07)
    - ### [M-08. Unchecked empty `_lockIds` array in `executeQueuedOperations` function](#M-08)
    - ### [M-09. GasLimit configuration vulnerability in CCIP message building, higher-than-necessary gas costs ](#M-09)
    - ### [M-10. Assumption of `sdlToken` at Zeroth Index in `destTokenAmounts`: Potential Unintended Consequences](#M-10)
- ## Low Risk Findings
    - ### [L-01. Potential front-running issue in `initialize`](#L-01)
    - ### [L-02. Address validation vulnerability in `getLockIdsByOwner` function](#L-02)
    - ### [L-03. Lack of zero address check in `setRewardsInitiator` function](#L-03)
    - ### [L-04. Zero-Balance check in `recoverTokens`](#L-04)
    - ### [L-05. Missing zero address validation in `RESDLTokenBridge` Constructor](#L-05)
    - ### [L-06. Redundant Loop in `_mintQueuedNewLocks`: Unnecessary Computations Leading to Extra Gas Consumption](#L-06)
    - ### [L-07. No validation for `_amount` in migrate function](#L-07)
    - ### [L-08. `_sender` is not validated in `onTokenTransfer` function](#L-08)
    - ### [L-09. Single-step ownership change introduces risks in `LinearBoostController`](#L-09)
    - ### [L-10. Fee token flexibility issue, has the potential to revert transactions](#L-10)
    - ### [L-11. Deprecated `safeApprove` OZ function is used, unintended reverts can happen](#L-11)
    - ### [L-12. Missing Event Emissions in SDLPool and SDLPoolCCIPController Admin Setter Functions](#L-12)
    - ### [L-13. SDLPool assumes `lockIdsFound` will always be equal to `lockCount`](#L-13)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: stake.link

### Dates: Dec 22nd, 2023 - Jan 12th, 2024

[See more contest details here](https://www.codehawks.com/contests/clqf7mgla0001yeyfah59c674)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 10
   - Low: 13


# High Risk Findings

## <a id='H-01'></a>H-01. Unauthorized withdrawals without unlock initiation check in `_executeQueuedLockUpdates`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L451-L510

## Summary

The `_executeQueuedLockUpdates` function in `SDLPoolSecondary` has a potential vulnerability that allows withdrawals without explicitly checking if the unlock period has been initiated using the `initiateUnlock` function. This omission could lead to unintended behavior and may result in undesired consequences for users interacting with the staking mechanism.

## Vulnerability Details

In the withdrawal section of the `_executeQueuedLockUpdates` function, the code initiates withdrawals based on the condition that `baseAmountDiff` is negative. However, it lacks an explicit check to ensure that the unlock period has been properly initiated using the `initiateUnlock` function. This could potentially allow users to withdraw funds without going through the required unlock process.

## Impact

The impact of this vulnerability is significant, as it may allow users to prematurely withdraw funds from their locks without adhering to the intended unlocking mechanism. This could undermine the logic of the staking contract and result in unexpected financial outcomes for both users and the contract itself.

## POC

Proof of Concept (PoC) Steps:

1. **Create a Lock by Staking SDL:**
   - User initiates the creation of a lock by staking SDL tokens.
   - This process involves calling a function, e.g., `createLock`, to establish a staking position with specified parameters like amount and locking duration.

2. **Update the Lock:**
   - After creating the lock, the user decides to update it by increasing the staked amount.
   - The user calls the update function, e.g., `updateLock`, which internally triggers `_queueLockUpdate` to queue the updated lock information in the `queuedLockUpdates` array.

3. **Execute Queued Operations:**
   - The user decides to execute queued operations by calling the external function `executeQueuedOperations` and passing the relevant `_lockIds`.

4. **Execute Queued Lock Updates (`_executeQueuedLockUpdates`):**
   - The `executeQueuedOperations` function calls the internal function `_executeQueuedLockUpdates`.
   - Within `_executeQueuedLockUpdates`, the function iterates through the queued updates for the specified lock using the `queuedLockUpdates` array.

5. **Withdrawal Without Unlock Initiation Check:**
   - During the iteration, when the condition `if (baseAmountDiff < 0)` is satisfied, a withdrawal operation is triggered.
   - Notably, there is no explicit check to verify whether the lock has been initiated for unlocking using the `initiateUnlock` function.

6. **Withdrawal Operation:**
   - The line `sdlToken.safeTransfer(_owner, uint256(-1 * baseAmountDiff));` is executed, facilitating the withdrawal of the staked amount to the user.

7. **Proof of Vulnerability:**
   - Since there is no explicit check for unlock initiation, the user is able to withdraw funds from the lock without initiating the unlock process through the `initiateUnlock` function.
   - This constitutes a vulnerability, as withdrawals should ideally be contingent on the proper initiation of the unlock period.


## Tools Used

Manual review.

## Recommendations

**Implement Check for Unlock Initiation:**
   Modify the withdrawal section of the `_executeQueuedLockUpdates` function to include an explicit check for the initiation of the unlock period. Ensure that withdrawals are only allowed if the unlock period has been properly initiated using the `initiateUnlock` function.

   ```solidity
   if (baseAmountDiff < 0) {
       // withdraw if negative
       if (lock.expiry == 0) {
           // Check if the unlock period is initiated
           revert UnlockNotInitiated();
       }

       // ... rest of the code
   }
   ```


## <a id='H-02'></a>H-02. Potential Division by zero in `distributeRewards` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L84


## Summary

The `distributeRewards` function in the `SDLPoolCCIPControllerPrimary` is susceptible to a division by zero error when `totalRESDL` is zero. This situation arises if there are no locks on any secondary chains, resulting in a potential runtime error during reward distribution.

## Vulnerability Details

The specific vulnerability lies in the following code snippet:

```solidity
for (uint256 j = 0; j < numDestinations; ++j) {
    uint64 chainSelector = whitelistedChains[j];

    uint256 rewards = j == numDestinations - 1
        ? tokenBalance - totalDistributed
        : (tokenBalance * reSDLSupplyByChain[chainSelector]) / totalRESDL;
    distributionAmounts[j][i] = rewards;
    totalDistributed += rewards;
}
```

If `totalRESDL` is zero, the division operation may result in a runtime error, leading to transaction revert.

## Impact

The rewards distribution function is a critical component of `SDLPoolCCIPControllerPrimary`. If a division by zero occurs, the potential error could significantly impact the correctness and security of the entire system, affecting the expected behavior of the reward distribution mechanism. This vulnerability may lead to a runtime error, specifically resulting from a division by zero, posing a substantial risk to the robust functioning of the contract and its associated interactions.

## Tools Used

Manual review.

## Recommendations

It is recommended to implement a check to ensure that `totalRESDL` is greater than zero before performing the division operation. This prevents division by zero errors and provides a more robust handling of scenarios where there are no locks on any secondary chains.

Here is an example of the recommended code modification:

```solidity
for (uint256 j = 0; j < numDestinations; ++j) {
    uint64 chainSelector = whitelistedChains[j];

    uint256 rewards;
    if (totalRESDL > 0) {
        rewards = j == numDestinations - 1
            ? tokenBalance - totalDistributed
            : (tokenBalance * reSDLSupplyByChain[chainSelector]) / totalRESDL;
    } else {
        rewards = 0; // Handle the case where totalRESDL is zero
    }

    distributionAmounts[j][i] = rewards;
    totalDistributed += rewards;
}
```

This modification ensures a safer distribution mechanism by preventing division by zero and provides better handling of edge cases.
## <a id='H-03'></a>H-03. Array length mismatch in `_buildCCIPMessage` function, may lead to index out of bounds errors            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L344-L367


## Summary

In `SDLPoolCCIPControllerPrimary` contract, an issue is present in the `_buildCCIPMessage` function. The function lacks validation for array length matching between `_tokens` and `_tokenAmounts`. This could potentially lead to unexpected behavior and runtime issues if the arrays provided as arguments have different lengths.

## Vulnerability Details

The `_buildCCIPMessage` function does not include a check to ensure that the lengths of the `_tokens` and `_tokenAmounts` arrays match. This oversight could result in array out-of-bounds errors, data inconsistency, and potential reentrancy vulnerabilities if the function is used with mismatched array lengths.

The issue is present in the `_buildCCIPMessage` function within the `SDLPoolCCIPControllerPrimary` contract.

```solidity
function _buildCCIPMessage(
    address _destination,
    uint256 _mintStartIndex,
    address[] memory _tokens,
    uint256[] memory _tokenAmounts,
    bytes memory _extraArgs
) internal view returns (Client.EVM2AnyMessage memory) {
    // Missing check for array length matching
    // Rest of the function...
}
```

## POC

In Solidity, arrays are zero-indexed, meaning the index of the first element is 0, the second element is at index 1, and so on. If you attempt to access an array element at an index beyond the array's declared length, you may encounter "Index out of bounds" errors. Let's break down how this issue can arise in the context of the our scenario:

Suppose we have the following arrays:

```solidity
address[] memory _tokens = new address[](5);
uint256[] memory _tokenAmounts = new uint256[](4);
```

Now, let's say you attempt to access the fifth element of `_tokenAmounts` within a loop or by directly referencing it:

```solidity
for (uint256 i = 0; i < _tokens.length; i++) {
    // Accessing the fifth element of _tokenAmounts
    uint256 amount = _tokenAmounts[i];
    // ...
}
```

In this loop, when `i` reaches 4 (the last valid index for `_tokenAmounts`), the attempt to access `_tokenAmounts[4]` would go beyond the declared length of `_tokenAmounts`, which is 4.

Since `_tokenAmounts` is declared with a length of 4, valid indices for this array are 0, 1, 2, and 3. Attempting to access an element at index 4 would result in an "Index out of bounds" error. Solidity does not automatically resize arrays or pad them with default values; it simply does not allow access to elements beyond the declared length.

This situation can lead to unexpected behavior, crashes, or contract reversion, making it crucial to validate array lengths before attempting to access their elements to prevent such errors in smart contract code.

## Impact

The absence of array length validation in the `_buildCCIPMessage` function poses a high-severity risk, potentially leading to inconsistent data processing and compromising the correctness and security of the entire system, especially in critical cross-contract interactions.

## Tools Used

Manual review.

## Recommendations

To address this issue and enhance the robustness of the `_buildCCIPMessage` function, we recommend the following:

**Add Array Length Validation:**
   - Implement a check at the beginning of the function to ensure that the lengths of `_tokens` and `_tokenAmounts` arrays match. Revert the transaction if a mismatch is detected.

   ```solidity
   require(_tokens.length == _tokenAmounts.length, "Array length mismatch between _tokens and _tokenAmounts");
   ```


## <a id='H-04'></a>H-04. Potential Reentrancy Vulnerability in `handleIncomingRESDL` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L125-L134


## Summary

The `handleIncomingRESDL` function in the SDLPoolCCIPControllerPrimary contract has a potential vulnerability related to reentrancy attacks. The function's state update, which involves reducing the total supply of a token on the source chain, is placed after an external call to `ISDLPoolPrimary(sdlPool).handleIncomingRESDL`. This order of operations may expose the contract to reentrancy attacks. Basically, it is not following the CEI pattern.

## Vulnerability Details

The vulnerability arises from the fact that the state update is not done atomically with the external call. In case of reentrancy, an attacker could potentially re-enter the function before the state is updated, leading to an inconsistent state and unintended consequences.

## Impact

If a reentrancy attack occurs, the reduction in the total supply of the token on the source chain may not take effect as intended. This could result in an inconsistent state and potential issues with the correctness and security of the contract.

## Tools Used

Manual review.

## Recommendations

To address the vulnerability, it is recommended to reorder the operations in the `handleIncomingRESDL` function. The state update, particularly the reduction in total supply, should be done before any external calls to ensure atomicity and prevent reentrancy vulnerabilities. Additionally, consider using reentrancy guards or mutex patterns to further enhance the security of the contract.

Updated code should look like this:

```diff
// Ensure state updates are performed before external calls to prevent reentrancy vulnerabilities
function handleIncomingRESDL(
    uint64 _sourceChainSelector,
    address _receiver,
    uint256 _tokenId,
    ISDLPool.RESDLToken calldata _reSDLToken
) external override onlyBridge {
+   // Perform state updates before the external call
+    reSDLSupplyByChain[_sourceChainSelector] -= _reSDLToken.amount + _reSDLToken.boostAmount;

    sdlToken.safeTransfer(sdlPool, _reSDLToken.amount);
    
    ISDLPoolPrimary(sdlPool).handleIncomingRESDL(_receiver, _tokenId, _reSDLToken);
}
```

## <a id='H-05'></a>H-05. Mismatched Arrays in `distributeRewards` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L56-L93

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L244-L287


## Summary

The `distributeRewards` function in `SDLPoolCCIPControllerPrimary` attempts to distribute rewards across secondary chains. However, it might suffers from a critical issue of creating two arrays, `tokens` and `distributionAmounts`, with mismatched lengths. The subsequent call to the `_distributeRewards` function using these arrays can lead to runtime errors and unexpected behavior.

## Vulnerability Details & POC

1. The `tokens` array is created with a length of 5 items.
2. The `whitelistedChains` array, representing the total whitelisted chains, is configured with a length of 4.
3. The `distributionAmounts` array is initialized with a length of `numDestinations` (whitelisted chains length), which is 4.

```solidity
uint256[][] memory distributionAmounts = new uint256[][](numDestinations);
```

4. Inside the loop, `distributionAmounts` is populated with `new uint256[](tokens.length)`, resulting in a `distributionAmounts` array with a length of 4.

5. When calling `_distributeRewards` in the final loop, the arrays passed to the function have mismatched lengths:

```solidity
_distributeRewards(whitelistedChains[i], tokens, distributionAmounts[i]);
```

6. The `_distributeRewards` function, in its loops, uses `if (_rewardTokenAmounts[i] != 0)` checks. Since `_rewardTokens` has a length of 5, and `_rewardTokenAmounts` is only 4, this can lead to out-of-bounds array access, resulting in runtime errors.

## Impact

This mismatch in array lengths can cause unexpected behavior, runtime errors, and potential failure of the distribution logic. It may compromise the integrity of the reward distribution mechanism across secondary chains.

## Tools Used

Manual review

## Recommendations

To address this issue, it is crucial to ensure that the lengths of `_rewardTokens` and `_rewardTokenAmounts` are always the same when calling the `_distributeRewards` function. This can be achieved by performing proper validation or ensuring that arrays are constructed with matching lengths.



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Unchecked same-chain transfer vulnerability, user can lose tokens            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/RESDLTokenBridge.sol#L84-L135

## Summary:

In the `RESDLTokenBridge` contract, the `transferRESDL` function facilitates the transfer of an reSDL token to a destination chain. However, a vulnerability exists where same-chain transfers are not explicitly checked. If the source chain is specified as the destination chain, the transfer will proceed without reversion.

## Vulnerability Details:

1. **Unchecked Same-Chain Transfer:**
   - The `transferRESDL` function does not include an explicit check to prevent same-chain transfers. As a result, the subsequent `handleOutgoingRESDL` call may inadvertently execute state changes intended for cross-chain transfers, leading to unintended behavior.

2. **Lock Deletion and Balance Update:**
   - In the `handleOutgoingRESDL` function, the deletion of the lock from the `locks` mapping, the adjustment of the sender's balance, and the update of effective balances are executed without verifying that the transfer is to a different chain.

3. **Potential Transfer to ccipController:**
   - In the same-chain transfer scenario, the `sdlToken` is transferred to the specified `_sdlReceiver`. However, due to the absence of a same-chain check, this transfer may inadvertently route the token to `ccipController`.

## Impact:

The unchecked same-chain transfer vulnerability introduces the following risks:

- Unintended state changes, including lock deletion and balance adjustments, during same-chain transfers.
- Potential transfer of tokens to `ccipController` when the source chain is erroneously specified as the destination chain.
- Loss of fees

## POC

**Same-Chain Transfer Mistake:**

1. **User Mistake:**
   - The user mistakenly specifies the same chain as both the source and destination.

   ```solidity
   RESDLTokenBridge.transferRESDL(SepoliaChainID, ReceiverAddress, TokenID);
   ```

2. **No Same-Chain Check:**
   - Currently, there is no check in the `transferRESDL` function to prevent same-chain transfers.

3. **Unintended Execution:**
   - The `handleOutgoingRESDL` function is called internally, expecting a cross-chain transfer.

4. **Lock Deletion:**
   - The lock associated with `TokenID` is deleted from the `locks` mapping.

   ```solidity
   delete locks[TokenID].amount;
   ```

5. **Balance Adjustment:**
   - The sender's balance is decreased by 1.

   ```solidity
   balances[_sender] -= 1;
   ```

6. **Effective Balances Update:**
   - The effective balances are adjusted: `_sender`'s decreased, `ccipController`'s increased.

   ```solidity
   effectiveBalances[_sender] -= totalAmount;
   effectiveBalances[ccipController] += totalAmount;
   ```

7. **Token Transfer:**
   - The `sdlToken` is transferred to the specified `_sdlReceiver`.

   ```solidity
   sdlToken.safeTransfer(_sdlReceiver, lock.amount);
   ```

8. **Unintended Result:**
   - The intended behavior was for cross-chain transfers, but due to the same-chain mistake, these actions happen within the same chain.

## Tools Used:

Manual code review.

## Recommendations:

**Add Same-Chain Transfer Check:**
   - Implement a check in the `transferRESDL` function to explicitly disallow same-chain transfers. This check should prevent the subsequent execution of state-changing operations for same-chain scenarios.


## <a id='M-02'></a>M-02. Incomplete token approval management in `handleOutgoingRESDL` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolPrimary.sol#L172-L199


## Summary

In SDLPoolPrimary function `handleOutgoingRESDL` related to the handling of "reSDL" locks and their outgoing transfers has no check for approval removal. The primary concern identified is the absence of a conditional check for approval deletion in the `handleOutgoingRESDL` function.

## Vulnerability Details

The `handleOutgoingRESDL` function processes outgoing transfers of "reSDL" locks to another chain. While the function correctly manages lock-related information and token transfers, it lacks a conditional check for approval deletion when the entire lock amount is transferred.

## Impact

The absence of the conditional check for approval deletion may result in leaving unnecessary token approvals in the state, potentially leading to inconsistent contract behavior. This could impact the security and efficiency of the contract.

Unnecessary token approvals can pose a security risk by allowing unauthorized contracts or users to spend tokens on behalf of the contract owner, as these approvals may remain valid indefinitely.

## Tools Used

- Manual code review

## Recommendations

**Conditional Check for Approval Deletion:**
 In the `handleOutgoingRESDL` function, introduce a conditional check similar to the one present in the `withdraw` function to delete token approvals when the entire lock amount is being transferred.

Example modification:
```solidity
// ...

sdlToken.safeTransfer(_sdlReceiver, lock.amount);

// delete token approvals if needed
if (totalAmount == lock.amount) {
    if (tokenApprovals[_lockId] != address(0)) {
        delete tokenApprovals[_lockId];
    }
}

// ...   
```


## <a id='M-03'></a>M-03. Array length mismatch vulnerability in `performUpkeep`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/RewardsInitiator.sol#L83-L94


## Summary

The `RewardsInitiator` contract containing the `performUpkeep` function. The review focused on potential vulnerabilities related to array lengths and out-of-bounds array access. A specific concern was identified where the lengths of arrays `strategies` and `strategiesToUpdate` were not explicitly checked for mismatches before accessing array indices in a loop.

## Vulnerability Details

The `performUpkeep` function may be susceptible to out-of-bounds array access if the lengths of arrays `strategies` and `strategiesToUpdate` do not match. The code lacks a check to ensure that the indices provided in `strategiesToUpdate` are within the valid range of the `strategies` array.

### Code Snippet:
```solidity
function performUpkeep(bytes calldata _performData) external {
    address[] memory strategies = stakingPool.getStrategies();
    uint256[] memory strategiesToUpdate = abi.decode(_performData, (uint256[]));

    // Check for a mismatch in array lengths
    if (strategiesToUpdate.length > strategies.length) {
        revert MismatchedArrayLengths();
    }

    if (strategiesToUpdate.length == 0) revert NoStrategiesToUpdate();

    for (uint256 i = 0; i < strategiesToUpdate.length; ++i) {
        if (IStrategy(strategies[strategiesToUpdate[i]]).getDepositChange() >= 0) revert PositiveDepositChange();
    }

    stakingPool.updateStrategyRewards(strategiesToUpdate, "");
}
```

## Impact

If a mismatch in array lengths occurs, the contract may attempt to access out-of-bounds indices in the `strategies` array, leading to a runtime error and causing the entire transaction to revert. This could result in unexpected behavior when Chainlink automation calls performUpkeep.

## POC

Scenario:
- `strategies`: 2 items.
- `strategiesToUpdate`: 3 items.

1. **First Iteration (`i = 0`):**
   - Check on `strategiesToUpdate[0]` succeeds as it points to a valid index in `strategies`.

2. **Second Iteration (`i = 1`):**
   - Check on `strategiesToUpdate[1]` succeeds as it points to a valid index in `strategies`.

3. **Third Iteration (`i = 2`):**
   - Attempt to check `strategiesToUpdate[2]` fails, as it points to an index outside the valid range of the 2-item `strategies` array.

**Result:**
- The third iteration attempts an out-of-bounds access, risking unexpected behavior and potential transaction revert.
## Tools Used

Manual code review.

## Recommendations

**Mismatched Array Length Check:** Implement a check to ensure that the lengths of arrays `strategies` and `strategiesToUpdate` match before proceeding to the loop. This check should be performed at the beginning of the `performUpkeep` function.

```solidity
// Check for a mismatch in array lengths
if (strategiesToUpdate.length > strategies.length) {
    revert MismatchedArrayLengths();
}
```




## <a id='M-04'></a>M-04. Potential dust accumulation issue in Reward Distribution function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L82-L84

## Summary

The `SDLPoolCCIPControllerPrimary` contract function `distributeRewards` implements a reward distribution mechanism across multiple chains. The concern raised is related to potential precision issues in the reward calculation due to the use of division, which may lead to dust accumulation. The last rewarder in the distribution process appears to receive more rewards than others due to this mechanism.

## Vulnerability Details

The primary vulnerability lies in the following code snippet:

```solidity
uint256 rewards = j == numDestinations - 1
                    ? tokenBalance - totalDistributed
                    : (tokenBalance * reSDLSupplyByChain[chainSelector]) / totalRESDL;
```

The division operation in the formula may result in dust accumulation, and this dust is given to the last rewarder, potentially causing an uneven distribution of rewards.

## Impact

The impact of this issue is that the last rewarder in the distribution process may receive more rewards than expected due to precision loss from dust accumulation. While the dust amounts are likely to be small, this could still be a concern for fairness in reward distribution.

## POC


Let's assume this  **Scenario:** 
  
- **Given Data:**
  - Total tokenBalance: 110
  - Expected reward per destination: 27.5
  - Total RESDL supply (totalRESDL): 4

- **Calculation:**
  - Ideal reward per destination: 110 / 4 = 27.5

- **Issue:**
  - Solidity's handling of arithmetic operations with unsigned integers truncates the fractional part.
  - Result: Each destination receives the truncated value of 27 instead of the expected 27.5.

- **Impact:**
  - The remaining fractional rewards (0.5 each) accumulate as dust.
  - The last rewarder receives the accumulated dust, resulting in a higher total reward. ( tokenBalance - totalDistributed )

- **Proof of Concept:**
  - Total tokenBalance: 110
  - Calculated reward per destination (truncated): 27
  - Accumulated dust (truncated fractional parts): 0.5 + 0.5 + 0.5 = 1.5
  - Last rewarder's reward: 27 + 1.5 = 28.5

- **Conclusion:**
  - The Proof of Concept demonstrates that the last rewarder receives more rewards (28.5) than others due to the truncation of fractional parts in the reward calculation, leading to dust accumulation.

## Tools Used

Manual code review.

## Recommendations

Implement a rounding mechanism in the reward calculation to ensure that fractional parts are rounded up or down appropriately. This will prevent truncation-related precision issues, promote a fair distribution of rewards, and mitigate the accumulation of dust in the last rewarder's allocation.

## <a id='M-05'></a>M-05. Missing check for maximum locking duration in migrate function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolPrimary.sol#L256-L272

## Summary

In the SDLPoolPrimary smart contract, the `migrate` function includes a comment suggesting that it reverts if `_lockingDuration` exceeds a maximum limit. However, there is no explicit check for this condition in the code. This omission may lead to unexpected behavior if there is a requirement to enforce a maximum locking duration.

## Vulnerability Details

The `migrate` function in the SDLPoolPrimary contract lacks an explicit check for the maximum locking duration, as indicated by a comment in the code. Without this check, the contract may not enforce the intended constraint on the locking duration, potentially leading to unexpected behavior.

### Code Snippet
```solidity
function migrate(address _sender, uint256 _amount, uint64 _lockingDuration) external {
    if (msg.sender != delegatorPool) revert SenderNotAuthorized();
    sdlToken.safeTransferFrom(delegatorPool, address(this), _amount);
    _storeNewLock(_sender, _amount, _lockingDuration);
}
```

## Impact

This issue could lead to unexpected behavior, potentially allowing stakeholders to migrate stakes with locking durations exceeding the intended maximum. This might result in a deviation from the contract's expected behavior and compromise the security of the system.

## Tools Used

Manual review.

## Recommendation
Include an explicit check for the maximum locking duration inside the `migrate` function to ensure that the contract adheres to the specified constraints. For example:

```solidity
if (_lockingDuration > MAX_LOCKING_DURATION) revert InvalidLockingDuration();
```

Replace `MAX_LOCKING_DURATION` with the actual maximum locking duration allowed.
## <a id='M-06'></a>M-06. Potential risks in `RESDLTokenBridge` extraArgs configuration            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/RESDLTokenBridge.sol#L100

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/RESDLTokenBridge.sol#L231

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/RESDLTokenBridge.sol#L161-L164

## Summary

The `RESDLTokenBridge` contract uses `extraArgs` in the `setExtraArgs` and `transferRESDL` functions. The concern lies in the potential misconfiguration of `extraArgs` for specific chains, leading to the use of default values in the `_buildCCIPMessage` function. This may result in unintended behavior, non-optimal gas usage, and security risks.

## Vulnerability Details

The contract allows for the mutable updating of `extraArgs` using the `setExtraArgs` function. However, if `extraArgs` are not configured for a particular chain in the `transferRESDL` function, the `_buildCCIPMessage` function utilizes default values, including a non-refundable gas limit of 200,000. This lack of explicit configuration may lead to unexpected behavior, gas inefficiencies, and potential security risks. The order of message will also be effected because the default `strict` value will be false in extraArgs.

### Code Snippets

#### Relevant Code in `setExtraArgs`
```solidity
function setExtraArgs(uint64 _chainSelector, bytes calldata _extraArgs) external {
    extraArgsByChain[_chainSelector] = _extraArgs;
}
```

#### Relevant Code in `transferRESDL`
```solidity
function transferRESDL(uint64 _destinationChainSelector, /* other parameters */) external {
    // ...
    bytes memory extraArgs = extraArgsByChain[_destinationChainSelector];
    Client.EVM2AnyMessage memory evm2AnyMessage = _buildCCIPMessage(/* parameters */, extraArgs);
    // ...
}
```

#### Relevant Code in `_buildCCIPMessage`
```solidity
extraArgs: _extraArgs, // If not configured, this may use default values including a gas limit of 200,000.
```

## Impact

The impact of this issue includes:

1. **Unintended Behavior:** Default values may lead to unexpected behavior in the execution of CCIP messages on the destination chain.

2. **Non-Optimal Gas Usage:** The default gas limit of 200,000 may not be sufficient, resulting in transaction failures due to out-of-gas errors.

3. **Inconsistent Execution:** The order of parameters within the CCIP message may be unexpected, potentially causing inconsistent execution on the destination chain.

4. **Non-Refundable Gas Fees:** Default gas limit being non-refundable may result in higher gas fees being consumed without successful execution.

5. **Security Risks:** Depending on the specifics of the CCIP message, there may be security risks associated with unintended behavior or unexpected states.

## Tools Used

Manual review.

## Recommendations

1. **Documentation and Guidance:** Clearly document and provide guidance on the expected configuration of `extraArgs` for each supported chain to prevent unintentional misconfigurations.

2. **Default Value Considerations:** Carefully choose default values in the absence of configured `extraArgs` to ensure they align with the expected behavior of the system and provide optimal gas usage.

3. **Validation in `_buildCCIPMessage`:** Implement additional validation or checks in the `_buildCCIPMessage` function to handle scenarios where `extraArgs` are not configured, ensuring that default values are appropriate for the given context. 

Addressing these recommendations is crucial to mitigating the potential risks associated with the misconfiguration of `extraArgs` and ensuring the reliability and security of the `RESDLTokenBridge` contract.
## <a id='M-07'></a>M-07. Potential minting disruption and duplicate entry addition in `SDLPoolSecondary` Contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L336-L347

## Summary
The SDLPoolSecondary contract has a vulnerability where the `_mintStartIndex` parameter in the `handleIncomingUpdate` function is not validated for zero values. If an attacker manipulates the `_mintStartIndex` value by passing zero from the primary chain, it can lead to unnecessary array items with zero values being added, resulting in duplicate entries and potential gas waste. 

Additionally, there is a risk of reverting when the same user calling `handleIncomingUpdate` with a new `_mintStartIndex` after `updateInProgress` has been set to 0. 

## Vulnerability Details
The vulnerability arises from the inclusion of `_mintStartIndex` without proper validation in the `currentMintLockIdByBatch` array. If `_mintStartIndex` is zero, it can be added to the array, influencing the minting process and causing the addition of many duplicate entries with zero values. This lack of validation extends from the Primary Chain, where `_mintStartIndex` is not checked for zero values when constructing the CCIP message, to the Secondary Chain's `handleIncomingUpdate` function, allowing zero values to be added to the array during incoming updates.

And another issue is that incase if someone mistakenly pass `_mintStartIndex` as zero and the state of `updateInProgress` is updated to zero, so this user call will be rejected at second time because the state is updated already!


## Impact
The issue may result in the addition of numerous duplicate entries with zero values in the `currentMintLockIdByBatch` array, leading to inefficiencies, potential storage bloat, and disruptions in the minting process.

There is a risk of reverting when attempting to call `handleIncomingUpdate` with a new `_mintStartIndex` after `updateInProgress` has been set to 0.

## Tools Used
Manual review.

## Recommendations
1. Implement thorough validation checks on input parameters, specifically `_mintStartIndex`, both when constructing CCIP messages on the Primary Chain and when processing incoming updates on the Secondary Chain. Ensure that zero values are appropriately handled to prevent unintended consequences and the addition of duplicate entries in the minting logic.
2. Consider using require statements or other validation mechanisms to ensure that only valid inputs are accepted, enhancing the robustness and security of the minting process.

e.g Add the below check on all chains contract code where the start index validation is needed:

```
// Validate that _mintStartIndex is not zero before processing
require(_mintStartIndex != 0, "Invalid _mintStartIndex");
``` 

## <a id='M-08'></a>M-08. Unchecked empty `_lockIds` array in `executeQueuedOperations` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L247-L250

## Summary

In the `SDLPoolSecondary` contract, the `executeQueuedOperations` function allows for the possibility of passing an empty array (`_lockIds`) without any validation. Consequently, this leads to an unnecessary invocation of the `_executeQueuedLockUpdates` function, which, in turn, triggers the `updateRewards(_owner)` modifier. This modifier involves a loop to update rewards for a given account, resulting in significant unnecessary computations and gas wastage. The absence of validation for an empty `_lockIds` array can adversely impact users by causing unnecessary financial losses due to excessive gas consumption.

## Vulnerability Details

The vulnerability arises from the oversight in not checking for an empty `_lockIds` array within the `executeQueuedOperations` function. This oversight leads to the execution of `_executeQueuedLockUpdates` with an empty array, causing unnecessary computations within the `updateRewards(_owner)` modifier.

## Impact

The impact of this vulnerability is twofold. Firstly, it results in unnecessary gas consumption due to the execution of computations with an empty array. Secondly, users may incur additional financial losses as a consequence of this inefficient gas usage.

## Tools Used

Manual review

## Recommendations

It is recommended to implement a validation check within the `executeQueuedOperations` function to ensure that the `_lockIds` array is not empty before proceeding with the execution of `_executeQueuedLockUpdates`. This check will prevent unnecessary computations and gas wastage when there are no lock IDs to process. Additionally, consider revising the design to avoid triggering the `updateRewards(_owner)` modifier when it is not required, further optimizing gas usage and minimizing the risk of financial losses for users.

e.g you can add a check like this in `executeQueuedOperations` function

```diff
function executeQueuedOperations(uint256[] memory _lockIds) external {
+    if (_lockIds.length > 0) {
        _executeQueuedLockUpdates(msg.sender, _lockIds);    
+    }
    _mintQueuedNewLocks(msg.sender);
}
```

## <a id='M-09'></a>M-09. GasLimit configuration vulnerability in CCIP message building, higher-than-necessary gas costs             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L210-L228

## Summary:

The `gasLimit` parameter is used in building CCIP messages. However, when the `extraArgs` parameter is left empty ("0x"), a default gasLimit of `200,000` is set. This default value might lead to unnecessary gas consumption, especially when sending tokens to an externally owned account (EOA), and should be reconsidered.

## Vulnerability Details:

1. **Default Gas Limit:**
   - **Issue:** When extraArgs is set to "0x," the function sets a default gasLimit of 200,000. This might result in excess gas usage, especially when sending tokens to an EOA where `ccipReceive()` is not involved.
   - **Recommendation:** Consider adjusting the default gasLimit to 0 when sending tokens directly to an EOA, as no ccipReceive() implementation is called.

## Impact:

The current implementation might lead to higher-than-necessary gas costs, particularly when sending tokens to an EOA, potentially affecting the efficiency and cost-effectiveness of the cross-chain token transfer process.

## Tools Used:

Manual code review.

## Recommendations:

**GasLimit Flexibility:**
   - Consider setting the default gasLimit to `0` when sending tokens directly to an EOA without involving `ccipReceive()`.

**Sender Contract Best Practices**

For production code, adhere to the following best practices:

1. **Avoid Hardcoding `extraArgs`:** It is recommended to ensure that `extraArgs` is mutable. Implementing this flexibility allows for building `extraArgs` off-chain and passing it in function calls or storing it in a storage variable that can be updated as needed. By doing so, you maintain backward compatibility for potential future CCIP upgrades. Notably, your protocol already incorporates this functionality through `setExtraArgs`; therefore, prefer passing the actual `extraArgs` value instead of using hardcoded values in `_buildCCIPMessage` function.

Check more detail at https://docs.chain.link/ccip/getting-started

## <a id='M-10'></a>M-10. Assumption of `sdlToken` at Zeroth Index in `destTokenAmounts`: Potential Unintended Consequences            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/base/SDLPoolCCIPController.sol#L102-L110

## Summary
The `ccipReceive` function in the `SDLPoolCCIPController` is designed to forward messages to `reSDLTokenBridge` based on assumptions about the destination tokens. However, the code makes an assumption that the zeroth index of `destTokenAmounts[0].token` will always be equal to `address(sdlToken)`. This assumption could lead to potential issues if the logic evolves and more cases are introduced in the future.

## Vulnerability Details
The function currently checks if there is only one destination token, and if that token is `sdlToken`, it forwards the message to `reSDLTokenBridge`. This rigid handling assumes a fixed structure in the `destTokenAmounts` array, specifically that `sdlToken` will always be at the zeroth index. If this assumption is violated, unintended consequences could arise.

## Impact
The impact of this issue is moderate. If the assumption about the zeroth index is not maintained, the condition checking for `sdlToken` at the zeroth index may not hold, and the function could behave unexpectedly. This could potentially lead to incorrect handling of destination tokens.

## Tools Used
Manual review.

## Recommendations
**Flexible Handling:** Modify the function to handle multiple destination tokens in a more flexible and extensible way. Implement a `loop` to iterate through `destTokenAmounts` and handle each token individually.

**Documentation:** Clearly document the logic and assumptions in the code, especially if there are specific expectations regarding the zeroth index of `destTokenAmounts`. Ensure that future developers or maintainers understand the intended behavior and are aware of any assumptions made.

# Low Risk Findings

## <a id='L-01'></a>L-01. Potential front-running issue in `initialize`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L66-L79

## Summary

The contract `SDLPoolSecondary` utilizes an initialize function for upgradeability purposes, but the public accessibility of this function poses a security risk. Specifically, if the contract is not initialized in the same transaction as its construction, it opens the possibility of front-running attacks by malicious actors. 

## Vulnerability Details

The vulnerability lies in the public accessibility of the `initialize` function in the `SDLPoolSecondary` contract. While the contract is designed for upgradeability, allowing arbitrary or malicious values to be passed to the initialize function creates a potential security loophole. The risk is further heightened when the initialization does not occur in the same transaction as the contract construction, exposing legitimate actors to front-running attacks by malicious entities.

## Impact

The impact of this vulnerability could be severe, potentially leading to unauthorized modifications of the contract state or unintended behavior. Front-running attacks could compromise the integrity of the contract and negatively affect the project's functionality. The security of user funds and the overall reliability of the project may be jeopardized if this issue is not promptly addressed.

## Tools Used

- Manual review.

## Recommendations

Consider initializing contracts within the same transaction as their construction to be a priority in the design of the upgrade scheme and deployment mechanisms for this project. Alternatively, consider limiting who can call the `initialize` function.

## <a id='L-02'></a>L-02. Address validation vulnerability in `getLockIdsByOwner` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/base/SDLPool.sol#L177-L194


## Summary

The `getLockIdsByOwner` function has been audited for potential vulnerabilities and security concerns. The function is designed to retrieve lock IDs associated with a specific owner address. The audit identified a potential issue related to address validation and an assert statement within the function.

## Vulnerability Details

#### Lack of Address Validation

The function does not validate the `_owner` address parameter, which could result in unexpected behavior if the zero address is passed. This lack of validation can be exploited, as passing the zero address may manipulate the assert check inside the function.

The assert statement `assert(lockIdsFound == lockCount);` may be manipulated when the zero address is provided as the owner. In such cases, both `lockIdsFound` and `lockCount` will be zero, causing the assert statement to pass, potentially leading to unintended consequences.

## Impact

The impact of these vulnerabilities is significant. An attacker could exploit the lack of address validation to pass the zero address as the owner, manipulating the assert statement and potentially causing unexpected behavior in the contract.

## POC

- Copy the below function
- Run the test via `forge test --match-test testGetLockIdsByOwner -vvv`
- You will get the below results

Results:

```
Running 1 test for test/fuzz/FuzzTester.t.sol:FuzzTester
[PASS] testGetLockIdsByOwner() (gas: 15576)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.84ms
```

Test code:
```solidity
function testGetLockIdsByOwner() public {
    address zeroAddy = makeAddr("0x0");
    uint256[] memory _lockIds = sdlPool.getLockIdsByOwner(zeroAddy);
    uint256[] memory emptyArray;
    assertEq(_lockIds, emptyArray);
}
```
## Tools Used

Manual code review.

## Recommendations

**Address Validation:**
   - Implement address validation within the `getLockIdsByOwner` function to ensure that the provided address is not the zero address and is a valid Ethereum address.

## <a id='L-03'></a>L-03. Lack of zero address check in `setRewardsInitiator` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L234-L236


## Summary

In the `SDLPoolCCIPControllerPrimary` contract, the `setRewardsInitiator` function currently lacks validation for the rewards initiator address, allowing the input to potentially be set as the zero address. This absence of validation introduces security risks and may result in a loss of control over the rewards system.

## Vulnerability Details

The `setRewardsInitiator` function allows setting the rewards initiator address without checking if it is the zero address (`address(0)`). This lack of validation could pose security risks.

## Impact

- Unintended behavior: Allowing the zero address as the rewards initiator may result in unexpected behavior and consequences within the contract logic.
- Loss of control: Allowing the zero address as the rewards initiator could potentially compromise the sole authority to update rewards, leading to unintended control by unauthorized entities.

## Tools Used

Manually.

## Recommendations

To address the identified issue and enhance the security of the contract, the following recommendations are provided:

 **Add Zero Address Check:**
   - Implement a check in the `setRewardsInitiator` function to ensure that the provided rewards initiator address is not the zero address (`address(0)`).

   ```solidity
   function setRewardsInitiator(address _rewardsInitiator) external onlyOwner {
       require(_rewardsInitiator != address(0), "Invalid rewards initiator address");
       rewardsInitiator = _rewardsInitiator;
   }
   ```

   This modification ensures that only valid addresses are accepted as the rewards initiator.


## <a id='L-04'></a>L-04. Zero-Balance check in `recoverTokens`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L140-L147


## Summary

The `recoverTokens` function currently lacks a check for the token balance before initiating transfers. The absence of this check results in unnecessary gas consumption when attempting to transfer zero tokens.

## Impact

The absence of a zero-balance check may lead to unnecessary gas consumption when attempting to transfer tokens, even when the balance is zero. The recommended enhancement improves gas efficiency by ensuring transfers are executed only when the token balance is greater than zero.

## Tools used
Manual review.

## Recommendation Implementation

To address this issue, it is advised to incorporate the provided code snippet into the `recoverTokens` function. This ensures a more gas-efficient execution of token transfers, avoiding unnecessary transaction costs when the token balance is zero.

```solidity
uint256 balance = tokenToTransfer.balanceOf(address(this));
if (balance > 0) {
    tokenToTransfer.safeTransfer(_receiver, balance);
}
```

Additionally, enhance efficiency by replacing the current if condition check in `recoverTokens` with the require statement:

```solidity
require(_receiver != address(0), "InvalidReceiver");
```


## <a id='L-05'></a>L-05. Missing zero address validation in `RESDLTokenBridge` Constructor            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/RESDLTokenBridge.sol#L59-L69


## Summary

The `RESDLTokenBridge` contract is designed to handle the transfer of reSDL NFTs between primary and secondary chains. There is a potential vulnerability  in the constructor, where it does not validate parameters for zero address checks.

## Vulnerability Details

The constructor of the `RESDLTokenBridge` contract does not perform zero address checks for the input parameters (_linkToken, _sdlToken, _sdlPool, _sdlPoolCCIPController). This could lead to unintended issues, such as initializing the contract with invalid or zero addresses.

### Code Snippet
```solidity
constructor(address _linkToken, address _sdlToken, address _sdlPool, address _sdlPoolCCIPController) {
    linkToken = IERC20(_linkToken);
    sdlToken = IERC20(_sdlToken);
    sdlPool = ISDLPool(_sdlPool);
    sdlPoolCCIPController = ISDLPoolCCIPController(_sdlPoolCCIPController);
}
```

## Impact

If the constructor is called with zero or invalid addresses, it could result in unexpected behavior and potential vulnerabilities in the contract. This may lead to a compromise of the bridge's functionality and pose a risk to the security of the overall system.

## Tools Used

Manual review.

## Recommendations

**Zero Address Validation:** Add explicit zero address validation checks in the constructor for all input parameters to ensure that the contract is not initialized with invalid addresses.
   ```solidity
   require(_linkToken != address(0), "Invalid LINK token address");
   require(_sdlToken != address(0), "Invalid SDL token address");
   require(_sdlPool != address(0), "Invalid SDL Pool address");
   require(_sdlPoolCCIPController != address(0), "Invalid SDL Pool CCIP Controller address");
   ```

**Input Validation Standardization:** Consider implementing a standardized input validation approach across the contract to enhance overall security and reduce the risk of potential vulnerabilities.

It is recommended to address these issues promptly to enhance the security and reliability of the `RESDLTokenBridge` contract.
## <a id='L-06'></a>L-06. Redundant Loop in `_mintQueuedNewLocks`: Unnecessary Computations Leading to Extra Gas Consumption            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L384-L419

## Summary

The `SDLPoolSecondary` contract `_mintQueuedNewLocks` function has an optimization opportunity. The current implementation uses two loops to process and clean up an array of new locks. By introducing a minor adjustment, the need for the second loop is eliminated, resulting in improved efficiency and gas cost savings.

## Vulnerability Details

The original code contains a `while` loop that iterates through an array of new locks. Within the loop, certain elements are skipped using a `break` statement based on a condition (`newLockPointer.updateBatchIndex > finalizedBatchIndex`). The skipped elements were previously handled in a second loop, copying the remaining elements to the beginning of the array. Which can introduce many gas issues and redundant code.

## Impact

The impact of this issue lies in the efficiency and gas cost of the `_mintQueuedNewLocks` function. The redundant second loop for cleanup introduces unnecessary computational steps and increases gas consumption.

## Tools Used

Manual review.

## Recommendations

The proposed optimization involves moving the `++i` increment operation above the if condition and adding `continue` instead of `break`. This modification eliminates the need for the second loop, streamlining the code execution. The revised code snippet is as follows:

```solidity
while (i < numNewLocks) {
    NewLockPointer memory newLockPointer = newLocksByOwner[_owner][i];
    ++i; // Move increment operation above the continue statement
    if (newLockPointer.updateBatchIndex > finalizedBatchIndex) {
        continue; // Skip processing elements that meet the condition
    }

    // ... (rest of the loop remains unchanged)
    // Adjustments to update the necessary state are retained here
}
```

```diff
-for (uint256 j = 0; j < numNewLocks; ++j) {
-    if (i == numNewLocks) {
-        newLocksByOwner[_owner].pop();
-    } else {
-        newLocksByOwner[_owner][j] = newLocksByOwner[_owner][i];
-        ++i;
-    }
-}
```

This adjustment enhances the efficiency of the function and reduces gas consumption by eliminating the need for a separate loop to clean up the array.

**NOTE:** The same issue is present in `_executeQueuedLockUpdates` function also, and the recommendation can be applied to this function.
## <a id='L-07'></a>L-07. No validation for `_amount` in migrate function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolPrimary.sol#L264-L272

## Summary

In the `SDLPoolPrimary` contract, the `migrate` function lacks a validation check for the `_amount` parameter. When `_amount` is zero, indicating that no SDL tokens are being staked or migrated, a lock with zero value is created. This could lead to unintended consequences, as creating locks with zero value may not align with the intended behavior of the contract. Implementing a check for non-zero values in `_amount` is recommended to prevent the creation of zero-value locks during migration.

## Vulnerability Details

In the `migrate` function, there is no explicit check for zero values in the `_amount` parameter. Consequently, when zero is passed as the _amount during migration, a lock with zero value is created. While this does not cause a revert, it might lead to unintended consequences, such as the creation of zero-value locks and potential resource allocation for these locks.

## Impact

Allowing zero values in the `_amount` parameter during migration can lead to the creation of zero-value locks, posing risks such as unnecessary gas costs, increased complexity in auditing and contract comprehension, and potential resource allocation for zero-value locks.

## Tools Used

Manual review

## Recommendations

Implement a check at the beginning of the migrate function to ensure that `_amount` is greater than zero. This can prevent the creation of zero-value locks.

```
if (_amount == 0) revert NonZeroAmountRequired();
```
## <a id='L-08'></a>L-08. `_sender` is not validated in `onTokenTransfer` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/WrappedTokenBridge.sol#L83-L96

## Summary

The `WrappedTokenBridge.sol` contract contains a potential vulnerability in the `onTokenTransfer` function, where it fails to validate the `_sender` address for the zero address. This oversight may result in reverting the transfer, but the subsequent event emitted at the end of the `_transferTokens` function could log inaccurate data, including the zero address as the sender.

## Vulnerability Details

The vulnerability lies in the `onTokenTransfer` function, which does not perform a zero address check on the `_sender` parameter before calling the `_transferTokens` function. This oversight may lead to a situation where the transfer reverts due to an invalid sender address, but the subsequent event emission incorrectly logs the zero address as the sender.

## Impact

The potential impact of this vulnerability includes:

1. **Loss of Information:** The event logs may contain inaccurate data, with the zero address erroneously recorded as the sender in `TokensTransferred` events.

2. **Reduced User Experience:** Users relying on event logs to track token transfers or protocol activity may experience confusion and difficulties in identifying the actual senders of transactions.

3. **Debugging Challenges:** Developers and auditors reviewing the contract may face challenges in debugging and auditing due to inaccurate event logs.

## Tools Used

Manual review.

## Recommendations

To address this vulnerability, the following recommendations are provided:

**Validate `_sender` for Zero Address:** In the `onTokenTransfer` function, implement a check to ensure that the `_sender` parameter is a valid, non-zero address before proceeding with the `_transferTokens` function. This validation can help prevent reverting transactions due to an invalid sender.


## <a id='L-09'></a>L-09. Single-step ownership change introduces risks in `LinearBoostController`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/LinearBoostController.sol#L10


## Summary

LinearBoostController.sol contract inherits from OpenZeppelin's Ownable and contains critical functions like `setMaxLockingDuration` and `setMaxBoost`. There is a concern that the current ownership transfer mechanism may expose the contract to the risk of unauthorized access, potentially compromising these critical functions.

## Vulnerability Details

#### Lack of Two-Step Ownership Transfer
 LinearBoostController inherits OpenZeppelin's Ownable, and the default `transferOwnership()` function is used. This introduces the risk of unauthorized or accidental ownership transfers, allowing an attacker to compromise critical functions.

## Impact

The lack of a two-step ownership transfer mechanism poses a risk. If the ownership of the `LinearBoostController` contract is compromised, an attacker could exploit critical functions like `setMaxLockingDuration` and `setMaxBoost`. This could lead to unintended changes in system parameters, affecting the boost mechanism and economic incentives for stakers.

Also, the default `transferOwnership` use raise the risk of unauthorized or accidental ownership transfers, allowing an attacker to compromise critical functions.


## Tools Used

Manual review

## Recommendations

 **Two-Step Ownership Transfer:**
   -  Implement a two-step ownership transfer mechanism for LinearBoostController. ( use this https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol )


## <a id='L-10'></a>L-10. Fee token flexibility issue, has the potential to revert transactions            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L363

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L186

## Summary

An issue was identified where `SDLPoolCCIPControllerPrimary` and `SDLPoolCCIPControllerSecondary` contracts hard-code LINK tokens as the fee token in the `_buildCCIPMessage` function. This design choice may lead to issues for users who do not possess `LINK` tokens.

## Vulnerability Details

 The `_buildCCIPMessage` function in both SDLPoolCCIPControllerPrimary and SDLPoolCCIPControllerSecondary contracts hard-codes LINK tokens as the fee token. If a user doesn't have enough LINK tokens to cover the fee required for the CCIP message, the corresponding operation that attempts to initiate the CCIP message will revert

## Impact

The current design may cause transactions to fail for users who lack LINK tokens, as the fee calculation is based on LINK. The lack of flexibility in fee payment options could impact user experience and participation in the CCIP mechanism.

## Tools Used

Manual review.

## Recommendations

**Fee Payment Flexibility:**
   - Allow users to choose between paying fees in `LINK` tokens or the native token by introducing a boolean flag or parameter (e.g., `_payNative`) in the `_buildCCIPMessage` function.
   - **Implementation:** Modify the `_buildCCIPMessage` function to include an option for native token fee payment.

e.g you can pass `_payNative ? address(0) : address(linkToken)` in `_buildCCIPMessage` like below 
```solidity
_buildCCIPMessage(_receiver, amountToTransfer, _payNative ? address(0) : address(linkToken));
```

And then in building ccip message you can set `feeToken: feeTokenAddress`. 



## <a id='L-11'></a>L-11. Deprecated `safeApprove` OZ function is used, unintended reverts can happen            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L193

## Summary

The `approveRewardTokens` function in `SDLPoolCCIPControllerPrimary` is currently utilizing the deprecated `safeApprove` function from OpenZeppelin. This deprecated function can lead to unintended reverts and potential issues with fund locking.

## Vulnerability Details

The usage of the deprecated `safeApprove` function is flagged as a concern due to the possibility of unintended reverts. The OpenZeppelin ERC20 `safeApprove()` function has been deprecated, and it's advised to replace it with safer alternatives like `safeIncreaseAllowance` or `safeDecreaseAllowance`.

## Impact

The impact of using the deprecated `safeApprove` function includes the risk of unintended reverts, potentially leading to the locking of funds. This can affect the functionality and reliability of the `approveRewardTokens` function.

## Tools Used

Manual code review, and OpenZeppelin issue #2219 ( https://github.com/OpenZeppelin/openzeppelin-contracts/issues/2219 ).

## Recommendations

It is recommended to replace the deprecated `safeApprove` function with safer alternatives, such as `safeIncreaseAllowance` or `safeDecreaseAllowance`, as suggested in the OpenZeppelin comments. This update ensures compatibility with modern best practices and avoids potential issues related to deprecated functionality.
## <a id='L-12'></a>L-12. Missing Event Emissions in SDLPool and SDLPoolCCIPController Admin Setter Functions            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/ccip/base/SDLPoolCCIPController.sol#L130-L140

https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/base/SDLPool.sol#L363-L383

## Summary

The SDLPool contract's `setBaseURI`, `setBoostController`, and `setCCIPController` functions, along with the `setMaxLINKFee` and `setRESDLTokenBridge` functions in the SDLPoolCCIPController contract, lack event emissions. Events play a crucial role in notifying users about critical configuration changes, providing transparency and awareness.

## Vulnerability Details

1. **SDLPool - setBaseURI, setBoostController, setCCIPController:**
   - **Issue:** These functions do not emit events.
   - **Recommendation:** Emit events like this (`BaseURIChanged`, `BoostControllerChanged`, `CCIPControllerChanged`) to inform users about relevant changes.

2. **SDLPoolCCIPController - setMaxLINKFee, setRESDLTokenBridge:**
   - **Issue:** These functions do not emit events.
   - **Recommendation:** Emit events like this (`MaxLINKFeeChanged`, `RESDLTokenBridgeChanged`) to notify users about the modified configurations.

## Impact

The absence of event emissions on critical configuration changes may lead to user confusion and unawareness of modifications. By implementing events, users can stay informed about updates, contributing to a more transparent and user-friendly protocol.

## Tools Used

Manual review

## Recommendations

1. Implement event emissions for the SDLPool functions (`setBaseURI`, `setBoostController`, `setCCIPController`) to enhance user awareness and transparency.

2. Implement event emissions for the SDLPoolCCIPController functions (`setMaxLINKFee`, `setRESDLTokenBridge`) to keep users informed about changes to critical configurations.
## <a id='L-13'></a>L-13. SDLPool assumes `lockIdsFound` will always be equal to `lockCount`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/base/SDLPool.sol#L177-L194

## Summary

The `getLockIdsByOwner` function in the `SDLPool` contract contains a potential issue related to the use of the `assert` statement to check the equality of `lockIdsFound` and `lockCount`. This `assert` statement is meant to ensure an internal invariant but is currently used for an external condition. This choice could lead to less user-friendly behavior in case of external calls and could impact the overall reliability of the contract. 

Along with this, assert use is discouraged in production and is only preffered in test environments mostly.
 
## Vulnerability Details

In the `getLockIdsByOwner` function:

```solidity
assert(lockIdsFound == lockCount);
```

The `assert` statement checks whether `lockIdsFound` is equal to `lockCount`. This check is meant to ensure an internal invariant, but the function is marked as `external`, meaning it can be called externally. If an external caller triggers the `assert` condition to be false, the entire transaction will revert, consuming all gas and rolling back state changes.

## Impact

The impact of this vulnerability includes:

1. **Potential loss of Gas:**
   - External callers invoking the function might face a situation where the entire transaction fails due to the `assert` condition, consuming all gas. This is less user-friendly and could result in unexpected behavior for users interacting with the contract.

2. **Possible Inconsistency in State:**
   - If external conditions lead to a discrepancy between `lockIdsFound` and `lockCount`, the function could revert the entire transaction, potentially leaving the contract in an inconsistent state.

## Tools Used

Manual Review

## Recommendations

**Replace `assert` with `require` in `getLockIdsByOwner`:**
   - Consider replacing the `assert` statement with a `require` statement for external conditions. This change will provide a more graceful handling of conditions and a better user experience.

```solidity
require(lockIdsFound == lockCount, "Number of lock IDs does not match expected count");
```





