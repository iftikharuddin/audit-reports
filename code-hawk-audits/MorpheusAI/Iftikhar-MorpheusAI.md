# MorpheusAI - Findings Report

Check results @ [CodeHawks](https://www.codehawks.com/contests/clrzgrole0007xtsq0gfdw8if)

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Create Pool in Mock Distribution is missing validations; allowing duplicates, wrong decreaseInterval value and payoutStart value](#H-01)
    - ### [H-02. `lzReceive` lacks a check for duplicate payloads, allowing for potential replay attack](#H-02)
    - ### [H-03. Lack of validation for duplicate pools in `createPool`, user might stake in duplicate pool](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Wrong comment in `getPeriodReward`](#M-01)
    - ### [M-02. `stETH` can be paused, potential revert of all calls using wrap function](#M-02)
    - ### [M-03. Router can't be updated once deployed](#M-03)
    - ### [M-04. Do not call  `lzEndpoint.send` directly, use `_lzSend`](#M-04)
- ## Low Risk Findings
    - ### [L-01. `editPool` allows admin to set pool `payoutStart` in past, potential issue of premature claim and withdrawal can occur](#L-01)
    - ### [L-02. Use custom gas in `sendMintMessage` instead of default gas](#L-02)
    - ### [L-03. Logical flaw in `manageUsersInPrivatePool` function, preventing correct handling of equal stake amounts](#L-03)
    - ### [L-04. Critical functions are missing event emission](#L-04)
    - ### [L-05. Lack of validation in `setRewardTokenConfig` function](#L-05)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: MorpheusAI

### Dates: Jan 30th, 2024 - Feb 3rd, 2024

[See more contest details here](https://www.codehawks.com/contests/clrzgrole0007xtsq0gfdw8if)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 4
   - Low: 5


# High Risk Findings

## <a id='H-01'></a>H-01. Create Pool in Mock Distribution is missing validations; allowing duplicates, wrong decreaseInterval value and payoutStart value            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/mock/DistributionV2.sol#L18-L20


## Summary
The `createPool` function in the mock `DistributionV2` contract lacks essential validation checks, posing potential risks related to the pool's payout start, decrease interval, and the prevention of duplicate pools. The absence of these checks could lead to unexpected behavior, disruptions, and complexities in managing the system. The recommended checks aim to enhance the robustness and security of the contract.

## Vulnerability Details
1. **Payout Start Validation Missing:**
   - The `createPool` function does not check whether `pool_.payoutStart` is set to a future timestamp. This absence allows the possibility of setting payout start to 0 or a past date.
  
2. **Decrease Interval Validation Missing:**
   - The function does not verify if `pool_.decreaseInterval` is greater than zero. This lack of validation can lead to unexpected behavior, especially if the contract performs calculations involving `decreaseInterval`.

3. **Duplicate Pool Check Missing:**
   - The function does not check for duplicate pools before adding them to the `pools` array. This absence may result in confusion and complexity in managing and maintaining the system, as duplicate pools may be inadvertently added.

4. Even there is no access control, so anyone can call this function.

## Impact
1. **Payout Start Validation Missing:**
   - The absence of a payout start validation check could allow users to set payout start to 0 or a past date. This may impact functions relying on payout start, potentially leading to unexpected behavior.

2. **Decrease Interval Validation Missing:**
   - Lack of validation for `decreaseInterval` may introduce vulnerabilities, impacting calculations and potentially causing transaction reverts or unexpected results.

3. **Duplicate Pool Check Missing:**
   - The absence of a duplicate pool check may lead to confusion and complexities in managing pools, as unintended duplicate entries could be added to the `pools` array.

4. Anyone can call this

## Tools Used
Manual review and analysis 

## Recommendations
1. **Payout Start Validation:**
   - Add the following check to ensure that `payoutStart` is set to a future timestamp:
     ```solidity
     require(pool_.payoutStart > block.timestamp, "DS: invalid payout start value");
     ```

2. **Decrease Interval Validation:**
   - Add the following check to ensure that `decreaseInterval` is greater than zero:
     ```solidity
     require(pool_.decreaseInterval > 0, "DS: invalid decrease interval");
     ```

3. **Duplicate Pool Check:**
   - Implement a check to ensure that duplicate pools are not added to the `pools` array. This can be achieved by verifying the uniqueness of pool attributes before appending a new pool.

4. Add access control. 


## <a id='H-02'></a>H-02. `lzReceive` lacks a check for duplicate payloads, allowing for potential replay attack            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/L2MessageReceiver.sol#L31-L40

## Summary:
The `lzReceive` function in the `L2MessageReceiver` contract lacks a check for duplicate payloads, allowing for potential replay attacks. This vulnerability could lead to unintended consequences, incorrect state changes, and resource exhaustion.

## Vulnerability Details:
Without a check for duplicate payloads, an attacker could replay a previously valid payload with the same nonce, resulting in undesired execution of the same transaction multiple times.

Duplicate payloads might trigger operations that are not designed to handle multiple executions, potentially leading to unexpected behaviors or unintended token minting.

## Impact:
The lack of a check for duplicate payloads poses a security and operational risk to the `L2MessageReceiver` contract, potentially leading to replay attacks, incorrect state changes, and unwanted operations.

## Recommendations:
Implement a nonce-based check inside the `lzReceive` function to prevent replay attacks. e.g you can add the following code snippet:

```solidity
/// @dev prevents layerzero relayer from replaying payload
mapping(uint16 => mapping(uint64 => bool)) public isValid;

if (isValid[srcChainId_][nonce_]) {
    revert Error.DUPLICATE_PAYLOAD();
}

isValid[srcChainId_][nonce_] = true;
```


## <a id='H-03'></a>H-03. Lack of validation for duplicate pools in `createPool`, user might stake in duplicate pool            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/15f12cb992d568693abc1f702ef4e081696bf96c/contracts/Distribution.sol#L73-L80

## Summary

The `createPool` function in the `Distribution` contract allows the addition of duplicate pools to the `pools` array without checking if the content is unique. While the function is controlled by the admin, the lack of validation for uniqueness could lead to potential issues if duplicate pools are mistakenly added. This could impact the user experience and introduce confusion in managing and maintaining the pools.

## Vulnerability Details

The vulnerability arises from the absence of a check for duplicate content when adding pools using the `createPool` function. The function appends a new pool to the `pools` array without verifying if a similar pool already exists.

## Impact

1 - Duplicate pools may lead to confusion and complexity in managing and maintaining the system. Administrators and developers may find it challenging to debug and modify duplicated pools effectively.

2 - Users interacting with the contract might experience inconsistent behavior if they stake or claim rewards from duplicated pools. This could affect the overall user experience and trust in the system.

## POC

- Copy paste the below code to your tests ( i am using foundry )
- Run the test via this command `forge test --match-test testCreatePool -vvvv`

Test:

```solidity
function testCreatePool() public {
    // create one pool
    vm.prank(address(distribution.owner()));
    distribution.createPool(myPool);

    // create 2nd pool with same data `myPool`
    vm.prank(address(distribution.owner()));
    distribution.createPool(myPool);

    // check if data in array is duplicate
    assertEq(distribution.hasDuplicate(myPool), true);
}
```

Result:

```
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.51ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual analysis and review.

## Recommendations

To enhance the `createPool` function, add a validation mechanism to check if the array being added is unique. 
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Wrong comment in `getPeriodReward`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/libs/LinearDistributionIntervalDecrease.sol#L42-L45


## Summary
In the `getPeriodReward` function of the `LinearDistributionIntervalDecrease` library, there is a discrepancy between the comment and the code implementation regarding the condition for returning 0 when `startTime_` is equal to `endTime_`. The comment suggests a condition of strictly greater than (`>`), while the code implementation checks for greater than or equal to (`>=`). To ensure clarity and consistency, it is recommended to clarify the desired behavior and reconcile the code with the comments.

## Vulnerability Details
In the provided code snippet:
```solidity
// Return 0 when calculation 'startTime_' is bigger than 'endTime_'.
if (startTime_ >= endTime_) {
    return 0;
}
```
The comment suggests returning 0 when `startTime_` is strictly greater than `endTime_`, while the code checks for greater than or equal to.

## Impact
The discrepancy between the comment and code may lead to confusion regarding the intended behavior of the function. Depending on the desired logic, this inconsistency could impact the understanding of the code's functionality.

## Recommendations
Clarify the desired behavior and reconcile the code with the comments. Depending on the intended logic, either update the comment to reflect the current code implementation or adjust the code to match the comment.
## <a id='M-02'></a>M-02. `stETH` can be paused, potential revert of all calls using wrap function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/mock/tokens/WStETHMock.sol#L22-L27


## Summary
In the `wrap` function of the WStETHMock contract, there is an absence of a check for the paused status of the `stETH` token before executing a transfer. As `stETH` can be paused, it introduces a vulnerability where all transfers would revert if the `wrap` function is called during the paused state of `stETH`. 

## Vulnerability Details
The `wrap` function in WStETHMock wraps 1 stETH to 1 wstETH without checking whether `stETH` is paused. The vulnerable code snippet is as follows:

```solidity
function wrap(uint256 stETHAmount_) external returns (uint256) {
    require(stETHAmount_ > 0, "wstETH: can't wrap zero stETH");
    _mint(msg.sender, stETHAmount_);
    stETH.transferFrom(msg.sender, address(this), stETHAmount_);
    return stETHAmount_;
}
```

## Impact
The impact of this issue is that if the `wrap` function is called while the `stETH` token is in a paused state, all transfers will revert, potentially leading to undesired outcomes and confusion for users interacting with the contract.

## Recommendations
Add a check for the paused status of `stETH` before executing transfers in the `wrap` function. This can be achieved by updating the interface and adding the following check:

```solidity
require(!stETH.isStopped(), "wstETH: transfer stopped");
```


## <a id='M-03'></a>M-03. Router can't be updated once deployed            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/L2TokenReceiver.sol#L31

## Summary
The `L2TokenReceiver` utilizes the `router` address as an argument during the initialization (`init`) process. The concern arises from the fact that the `init` function is typically executed once during deployment. This design choice limits the ability to update the `router` address dynamically after deployment, potentially hindering adaptability to future changes.

## Vulnerability Details
The contract's initialization function (`init`) accepts the `router` address as an argument, allowing flexibility during the deployment phase. However, once the contract is deployed and initialized, there is no mechanism provided to update the `router` address. As a result, changes to the Uniswap V3 router contract address would necessitate a redeployment of the entire contract.

## Impact
The absence of a dedicated function to update the `router` address post-deployment poses challenges in scenarios where the Uniswap V3 router contract needs replacement or updates. Despite the contract inheriting from `OwnableUpgradeable`, implying ownership control, the lack of a specific upgradability mechanism makes future upgrades challenging. This limitation could result in downtime and introduce additional operational complexities when adapting to changes in the Uniswap V3 router contract.

## Recommendations
Consider implementing a function that allows the owner of the contract to update the `router` address after deployment. This update mechanism should be designed with appropriate access controls, such as the `onlyOwner` modifier, to ensure that only authorized entities can modify the address.

For example, adding a function like this will be good option:

```solidity
function setRouter(address newRouter) external onlyOwner {
    require(newRouter != address(0), "L2TR: invalid router address");
    router = newRouter;
}
```


## <a id='M-04'></a>M-04. Do not call  `lzEndpoint.send` directly, use `_lzSend`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/76898177fbedcbbf4b78b513d9fa151bbf3388de/contracts/L1Sender.sol#L130

## Summary
The `sendMintMessage` function in the `L1Sender` utilizes a direct call to `lzEndpoint.send` for cross-chain communication. This approach might introduce vulnerabilities and potential security risks. The recommended practice is to use the provided `_lzSend` function instead.

## Vulnerability Details
The direct use of `lzEndpoint.send` without utilizing the recommended `_lzSend` function can expose the contract to unforeseen issues and is not recommended to use it directly. ( https://layerzero.gitbook.io/docs/troubleshooting/layerzero-integration-checklist )

## Impact
Using direct calls to `lzEndpoint.send` includes potential vulnerabilities related to cross-chain communication. Along with this it doesn't check all the validation check which are present in `_lzSend` e.g _checkPayloadSize is missing. It may compromise the security and integrity of the contract, especially in scenarios where additional security checks or measures are implemented within the recommended `_lzSend` function.

## Recommendations
It is strongly advised to replace the direct call to `lzEndpoint.send` with the recommended `_lzSend` function in the `sendMintMessage` method. This adjustment will align with best security practices and help mitigate potential risks associated with cross-chain communication.

# Low Risk Findings

## <a id='L-01'></a>L-01. `editPool` allows admin to set pool `payoutStart` in past, potential issue of premature claim and withdrawal can occur            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/Distribution.sol#L82-L96

## Summary
In the `editPool` function of the Distribution contract, the vulnerability allows the admin to set the `payoutStart` parameter to a timestamp in the past, potentially enabling users to withdraw and claim rewards prematurely. The proposed fix involves introducing a validation check to ensure that `payoutStart` is always set to a future timestamp.

## Vulnerability Details
In the `editPool` function, the absence of a validation check allows the admin to manipulate the `payoutStart` parameter, potentially setting it to a timestamp in the past. This vulnerability has a significant impact on the contract's security, as users may exploit the lower `payoutStart` value to withdraw and claim rewards before the intended timeframe.

## Impact
The impact of this vulnerability is critical, as a lower or past value of `payoutStart` may lead to premature withdrawal and claiming of rewards by users. 

However, the likelihood is low. Because the edit is controlled by admin only.

## POC
- Copy the below test and run it via `forge test --match-test testEditPool -vvvv`
- In the result you can see when pool was created the `payoutStart: 86400` and when the edit was done it is lowered to `payoutStart: 56400`
```solidity
function testEditPool() public {
    // create a new private pool
    vm.prank(address(distribution.owner()));
    distribution.createPool(myPool); // poolId = 0
    vm.prank(address(distribution.owner()));
    distribution.editPool(0, myPoolEdit);
}
```

Result:
```
    ├─ [184054] Distribution::createPool(Pool({ payoutStart: 86400 [8.64e4], decreaseInterval: 86400 [8.64e4], withdrawLockPeriod: 43200 [4.32e4], claimLockPeriod: 43200 [4.32e4], withdrawLockPeriodAfterStake: 86400 [8.64e4], initialReward: 100000000000000000000 [1e20], rewardDecrease: 20000000000000000000 [2e19], minimalStake: 10000000000000000000 [1e19], isPublic: true }))
    │   ├─ emit PoolCreated(poolId: 0, pool: Pool({ payoutStart: 86400 [8.64e4], decreaseInterval: 86400 [8.64e4], withdrawLockPeriod: 43200 [4.32e4], claimLockPeriod: 43200 [4.32e4], withdrawLockPeriodAfterStake: 86400 [8.64e4], initialReward: 100000000000000000000 [1e20], rewardDecrease: 20000000000000000000 [2e19], minimalStake: 10000000000000000000 [1e19], isPublic: true }))
    │   └─ ← ()
    ├─ [365] Distribution::owner() [staticcall]
    │   └─ ← 0x0000000000000000000000000000000000000000
    ├─ [0] VM::prank(0x0000000000000000000000000000000000000000)
    │   └─ ← ()
    ├─ [35540] Distribution::editPool(0, Pool({ payoutStart: 56400 [5.64e4], decreaseInterval: 86400 [8.64e4], withdrawLockPeriod: 43200 [4.32e4], claimLockPeriod: 43200 [4.32e4], withdrawLockPeriodAfterStake: 86400 [8.64e4], initialReward: 100000000000000000000 [1e20], rewardDecrease: 20000000000000000000 [2e19], minimalStake: 10000000000000000000 [1e19], isPublic: true }))
    │   ├─ emit PoolEdited(poolId: 0, pool: Pool({ payoutStart: 56400 [5.64e4], decreaseInterval: 86400 [8.64e4], withdrawLockPeriod: 43200 [4.32e4], claimLockPeriod: 43200 [4.32e4], withdrawLockPeriodAfterStake: 86400 [8.64e4], initialReward: 100000000000000000000 [1e20], rewardDecrease: 20000000000000000000 [2e19], minimalStake: 10000000000000000000 [1e19], isPublic: true }))
    │   └─ ← ()
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 13.15ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)

```
## Recommendations
Introduce a validation check within the `editPool` function. Specifically, consider adding the following line of code to ensure that `payoutStart` is always set to a timestamp greater than the current block timestamp:

```solidity
require(pool_.payoutStart > block.timestamp, "DS: invalid payout start value");
```


## <a id='L-02'></a>L-02. Use custom gas in `sendMintMessage` instead of default gas            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/Distribution.sol#L174

https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/L1Sender.sol#L136


## **Summary:**
A potential issue related to the handling of gas limits in the `sendMintMessage` function of the `L1Sender.sol` contract was found. Currently it utilizes a default gas limit of 200,000, potentially leading to suboptimal gas usage. To address this issue, it is recommended to implement the use of `adapterParams` to allow users to set a custom gas limit, providing more flexibility and cost optimization.

## **Vulnerability Details:**
The vulnerability lies in the `sendMintMessage` function of the `L1Sender.sol`, where a default gas limit of `200,000` will be used by default. This fixed gas limit may not be optimal for all transactions, potentially leading to either overpayment for unused gas or insufficient gas for complex transactions. By not allowing users to customize the gas limit, the contract misses an opportunity for gas optimization.

## **Impact:**
The impact of the current implementation is primarily related to potential suboptimal gas usage. In scenarios where transactions have varying gas requirements, users may incur unnecessary costs or face delays due to inadequate gas limits. The issue does not pose a direct security threat but affects the efficiency and cost-effectiveness of protocol transactions.

## **Tools Used:**
Manual review.

## **Recommendations:**

**Implement `adapterParams` for Custom Gas Limit:**
   - Introduce a parameter for `adapterParams` in the `sendMintMessage` function to allow users to set a custom gas limit.
   - Ensure proper encoding and decoding of `adapterParams` according to the LayerZero documentation.

E.g the code can be changed like this
```diff
function sendMintMessage(
    address user_, 
    uint256 amount_, 
    address refundTo_, 
    bytes calldata adapterParams
) external payable onlyDistribution {
    RewardTokenConfig storage config = rewardTokenConfig;

    bytes memory receiverAndSenderAddresses_ = abi.encodePacked(config.receiver, address(this));
    bytes memory payload_ = abi.encode(user_, amount_);

    ILayerZeroEndpoint(config.gateway).send{value: msg.value}( 
        config.receiverChainId, 
        receiverAndSenderAddresses_, 
        payload_, 
        payable(refundTo_), 
        address(0x0), 
+        adapterParams  // Pass adapterParams here
    );
}
```

and from Distribution claim function you can pass the custom settings in `adapterParams`

```solidity
// Example usage in Distribution.sol
L1Sender(l1Sender).sendMintMessage{value: msg.value}(user_, pendingRewards_, _msgSender(), adapterParams);
```
## <a id='L-03'></a>L-03. Logical flaw in `manageUsersInPrivatePool` function, preventing correct handling of equal stake amounts            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/Distribution.sol#L123-L145


## Summary
In the `manageUsersInPrivatePool` function within the Distribution.sol, there is a condition check for staking and withdrawal based on the conditions `deposited_ < amount_` and `deposited_ > amount_` respectively. However, when `deposited_` is equal to `amount_`, both conditions evaluate to false, resulting in no action being taken. Although this does not pose a direct security risk, it may lead to confusion for the admin, as the intended staking action is not processed. To enhance clarity, it is recommended to handle this scenario by emitting an event in the else block.

### Vulnerability Details
In the current implementation, the staking and withdrawal conditions are as follows:

```solidity
if (deposited_ < amount_) {
    _stake(user_, poolId_, amount_ - deposited_, currentPoolRate_);
} else if (deposited_ > amount_) {
    _withdraw(user_, poolId_, deposited_ - amount_, currentPoolRate_);
}
```

The issue arises when `deposited_` is equal to `amount_`. In such cases, both conditions are false, and neither the `_stake` nor `_withdraw` functions are called. This leads to a situation where the admin's intended action to stake the same amount is not processed, causing potential confusion.

## Impact
The impact of this issue is that admin attempting to stake the same amount as previously deposited are unable to do so. The current conditions do not account for this scenario, resulting in unexpected behavior. Admins interacting with the contract may face confusion due to the lack of feedback on the equality of the deposited amount and the target amount.

## POC

- Copy the below test and run it via cmd `forge test --match-test testManageUsersInPrivatePool -vvvv`
- In the test admin give shares to 3 user by staking. Now the same amount is added again by calling stake but this time neither the stake will happen nor withdrawal because the both conditions will get false when `deposited == amount_` 

Test:
```solidity
function testManageUsersInPrivatePool() public {
    // create a new private pool
    vm.prank(address(distribution.owner()));
    distribution.createPool(myPrivatePool); // poolId = 0

    usersArray1 = new address[](3);
    usersArray1[0] = makeAddr("privateInvestor1");
    usersArray1[1] = makeAddr("privateInvestor2");
    usersArray1[2] = makeAddr("privateInvestor3");

    amountsArray1 = new uint256[](3);
    amountsArray1[0] = 1000;
    amountsArray1[1] = 500;
    amountsArray1[2] = 1500;

    vm.prank(address(distribution.owner()));
    // lets try staking
    distribution.manageUsersInPrivatePool(0, usersArray1, amountsArray1);
    console.log(distribution.getUserStakedAmount(usersArray1[0]));
    console.log(distribution.getUserStakedAmount(usersArray1[1]));
    console.log(distribution.getUserStakedAmount(usersArray1[2]));

    amountsArray1[0] = 1000;
    amountsArray1[1] = 500;
    amountsArray1[2] = 1500;

    vm.prank(address(distribution.owner()));
    // lets try staking the same amount we staked earlier
    distribution.manageUsersInPrivatePool(0, usersArray1, amountsArray1);
    console.log(distribution.getUserStakedAmount(usersArray1[0]));
    console.log(distribution.getUserStakedAmount(usersArray1[1]));
    console.log(distribution.getUserStakedAmount(usersArray1[2]));
}
```
Result:
```
Running 1 test for test/fuzz/AITests.t.sol:AiTestsFuzzTester
[PASS] testManageUsersInPrivatePool() (gas: 614108)
Logs:
  1000
  500
  1500
  1000
  500
  1500
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.70ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)

```
## Recommendations
To address this issue, it is recommended to add an event in the else block to provide clarity and feedback in the case where `deposited_` is equal to `amount_`:

```solidity
// Event to emit when amounts are equal
event AmountsEqual(uint256 indexed poolId, address user, uint256 amount);
```

This event will inform the admin that the specified amounts were already equal, ensuring transparency and aiding in debugging and monitoring efforts. By emitting this event, the contract becomes more resilient to potential confusion and offers a clearer communication channel for admins interacting with the contract.
## <a id='L-04'></a>L-04. Critical functions are missing event emission            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/76898177fbedcbbf4b78b513d9fa151bbf3388de/contracts/L1Sender.sol#L45-L62


## Summary
The contract `L1Sender` lacks event emissions in several critical administrative functions, including `setDistribution`, `setRewardTokenConfig`, `setDepositTokenConfig`, `sendDepositToken`, and `sendMintMessage`. Events are essential for transparency, monitoring, and user feedback.

## Vulnerability Details

The following functions lack event emissions:

1. `setDistribution`
2. `setRewardTokenConfig`
3. `setDepositTokenConfig`
4. `sendDepositToken`
5. `sendMintMessage`

The absence of events in these functions limits the contract's visibility, traceability, and user feedback.

## Impact

External entities cannot observe critical state changes, reducing the contract's transparency.

Analyzing the contract's behavior becomes challenging without events, hindering monitoring and analysis efforts.

Users may not receive necessary feedback about transaction outcomes, potentially leading to confusion.

## Recommendations

**Include Event Emissions:** Introduce appropriate event emissions in functions such as `setDistribution`, `setRewardTokenConfig`, `setDepositTokenConfig`, `sendDepositToken`, and `sendMintMessage`.

**Enhance Transparency:** Improve contract transparency by providing clear and comprehensive event logs for external observers.

**User Feedback:** Emit events to provide users with timely and accurate feedback about transaction outcomes.



## <a id='L-05'></a>L-05. Lack of validation in `setRewardTokenConfig` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/76898177fbedcbbf4b78b513d9fa151bbf3388de/contracts/L1Sender.sol#L49-L51

## Summary
The `L1Sender` contract contains a vulnerability in the `setRewardTokenConfig` function, where the input `newConfig_` struct is not adequately validated. This oversight allows all values within `newConfig_` to be set to zero, including `receiverChainId` and `receiver` address. If `receiverChainId` is set to 0, it may cause issues in cross-chain communication, disrupting the intended functionality. Additionally, if the `receiver` address is set to zero, the `sendMintMessage` function will send funds to the zero address.

## Vulnerability Details
The vulnerable code lies in the `setRewardTokenConfig` function, which allows an admin to update the configuration without proper validation. The `newConfig_` struct includes the `gateway`, `receiver`, and `receiverChainId` parameters. Lack of validation opens up the possibility of setting `receiverChainId` to 0, potentially leading to cross-chain communication issues, and `receiver` to a zero address, resulting in fund loss.

## Impact

**Zero Receiver Address:**
   - If `receiver` is set to a zero address, the `sendMintMessage` function will send funds to the zero address. While Ethereum allows sending funds to the zero address, it's typically unintentional and results in irrecoverable fund loss.

**Zero Chain ID:**
   - If `receiverChainId` is set to 0, it may cause disruptions in cross-chain communication. The `send` function from `ILayerZeroEndpoint` requires a valid chain ID, and a chain ID of 0 might lead to undefined behavior or issues in LayerZero communication.

## POC

- Copy below test and run it via `forge test --match-test testSetRewardTokenConfig -vvv`

Test:
```solidity
function testSetRewardTokenConfig() public {
    vm.prank(address (l1Sender.owner()));
    l1Sender.setRewardTokenConfig(rewardTokenConfig); // consider the struct values set to 0, and zero addy accordingly
    assertEq(rewardTokenConfig.gateway, 0x0000000000000000000000000000000000000000);
    assertEq(rewardTokenConfig.receiver, 0x0000000000000000000000000000000000000000);
    assertEq(rewardTokenConfig.receiverChainId, 0);
}
```

Result:
```
Running 1 test for test/fuzz/AITests.t.sol:AiTestsFuzzTester
[PASS] testSetRewardTokenConfig() (gas: 21476)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.34ms
```

## Recommendations

- Validate that `receiver` is not set to a zero address to prevent unintentional fund loss.
- Ensure that `receiverChainId` is a non-zero and valid chain ID to avoid disruptions in cross-chain communication.




