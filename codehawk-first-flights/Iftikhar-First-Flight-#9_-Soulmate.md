# First Flight #9: Soulmate - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. `claimRewards` can be manipulated, user can get extra rewards](#H-01)
    - ### [H-02. Anyone can claim the airdrop, as long the claim period is passed](#H-02)
    - ### [H-03. Sanity checks missing from `writeMessageInSharedSpace`, allowing unlimited messages and prev messages are overriten](#H-03)
    - ### [H-04. Logical issue in `getDivorce` function, allows user to get divorce without having a soulmate](#H-04)
    - ### [H-05. initVault can be front-run, wrong `managerContract` can be deployed](#H-05)

- ## Low Risk Findings
    - ### [L-01. Deposit and withdrawals validation checks missing](#L-01)
    - ### [L-02. Wrong data emitted in events of `LoveToken`](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #9

### Dates: Feb 8th, 2024 - Feb 15th, 2024

[See more contest details here](https://www.codehawks.com/contests/clsathvgg0005yhmxmoe455mm)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 0
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. `claimRewards` can be manipulated, user can get extra rewards            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L50-L58

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L70-L99

## Summary

The `Staking` contract contains a vulnerability related to the claim rewards function, where user rewards are calculated based on the total staked amount at the time of claim. This vulnerability allows users to potentially exploit the system by depositing additional tokens just before claiming, resulting in inflated rewards.

## Vulnerability Details

In the staking contract's claim function, the rewards calculation is performed using the formula:

```solidity
uint256 amountToClaim = userStakes[msg.sender] * timeInWeeksSinceLastClaim;
```

However, the `userStakes[msg.sender]` value can be updated through the deposit function, allowing users to manipulate their rewards by depositing additional tokens just before claiming. So if claim period is reached, he can deposit more tokens and get extra rewards in claiming.

## Impact

This vulnerability has the potential to result in disproportionate rewards for users who deposit additional tokens right before claiming. Such manipulation could lead to an imbalance in the reward distribution and impact the fairness of the staking system.

## POC

- Copy below test and run it via ``
Test:
```solidity
function testManipulateClaimRewards() public {
    uint256 balancePerSoulmates = 5 ether;
    uint256 weekOfStaking = 5;
    _depositTokenToStake(balancePerSoulmates);

    vm.warp(block.timestamp + weekOfStaking * 1 weeks + 1 seconds);

    // lets add more tokens before claiming rewards ;)
    _depositTokenToStakeForTest(balancePerSoulmates);
    vm.prank(soulmate1);
    stakingContract.claimRewards();

    assertTrue(loveToken.balanceOf(soulmate1) == weekOfStaking * balancePerSoulmates + balancePerSoulmates);
    // initial rewards was 25000000000000000000 but when user add more before claiming and claim period is passed
    // the new reward will be 30000000000000000000
    console2.log(loveToken.balanceOf(soulmate1));
}
```
Result:
```
Logs:
  30000000000000000000

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.32ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommendations
**Implement Snapshot Mechanism:**
   - Introduce a snapshot mechanism to capture the user's staked amount at the beginning of each staking period. Use this snapshot value for reward calculations, ensuring consistency and preventing manipulation.


## <a id='H-02'></a>H-02. Anyone can claim the airdrop, as long the claim period is passed            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Airdrop.sol#L51-L89

## Summary
The `claim` function in the Airdrop contract has a vulnerability that allows users to claim LoveTokens without verifying Soulmate NFT ownership. Additionally, the current implementation lacks proper checks for the claim period, enabling potential malicious users to exploit the contract and claim LoveTokens without the necessary ownership requirements.

## Vulnerability Details
### Lack of Soulmate NFT Ownership Check

The `claim` function does not include a check to ensure that the caller owns a Soulmate NFT. Without this check, any address can call the function and claim LoveTokens, violating the intended behavior described in the function comment.

### Inadequate Claim Period Verification
The existing claim period check is insufficient. The current implementation relies on the calculation of the number of days since the last claim, but it does not ensure that enough time has passed since the user's Soulmate NFT creation. As a result, malicious users can exploit this loophole to claim LoveTokens even if they do not meet the necessary ownership criteria.

## Impact

Malicious actors could exploit these issues to claim LoveTokens without the required Soulmate NFT ownership, leading to financial losses and undermining the fairness of the distribution mechanism.

## POC

- Copy the below test and run it via cmd `forge test --match-test testClaimByHacker -vvvv`
```solidity
function testClaimByHacker() public {
    address hacker = makeAddr("hacker");
    uint256 initialBalance = loveToken.balanceOf(address(airdropVault));

    vm.warp(block.timestamp + 200 days + 1 seconds);
    vm.prank(hacker);
    airdropContract.claim();

    uint256 afterBalance = loveToken.balanceOf(address(airdropVault));

    assertGt(initialBalance, afterBalance);
    assertGt(loveToken.balanceOf(hacker), 0);
}
```

Result:

```
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.91ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommendations

1. **Implement Soulmate NFT Ownership Check:**
   Include a check in the `claim` function to ensure that the caller owns a Soulmate NFT before proceeding with the LoveToken claim process.

2. **Enhance Claim Period Verification:**
   Improve the claim period verification by considering the time since the user's Soulmate NFT creation rather than only the time since the last claim. This ensures that users can claim LoveTokens only once per day, in line with the intended functionality.

For example the new code should look like this so that it has the required checks:
```solidity
function claim() public {
    // Check if the caller owns a Soulmate NFT
    if (!soulmateContract.ownerOf(msg.sender)) {
        revert Airdrop__NotSoulmateNFTOwner();
    }

    // Check if the couple is divorced
    if (soulmateContract.isDivorced()) {
        revert Airdrop__CoupleIsDivorced();
    }

    // Calculate the number of days since the last claim
    uint256 numberOfDaysSinceLastClaim = (block.timestamp - lastClaim[msg.sender]) / daysInSecond;

    // Check if the claim period has passed (e.g., 1 day)
    if (numberOfDaysSinceLastClaim < 1) {
        revert Airdrop__ClaimFrequencyExceeded();
    }

    // Calculate the number of days the couple has been reunited
    uint256 numberOfDaysInCouple = (block.timestamp - soulmateContract.idToCreationTimestamp(soulmateContract.ownerToId(msg.sender))) / daysInSecond;

    // Calculate the claimable token amount
    uint256 amountAlreadyClaimed = _claimedBy[msg.sender];
    uint256 tokenAmountToDistribute = (numberOfDaysInCouple * 10 ** loveToken.decimals()) - amountAlreadyClaimed;

    // Dust collector
    if (tokenAmountToDistribute > loveToken.balanceOf(address(airdropVault))) {
        tokenAmountToDistribute = loveToken.balanceOf(address(airdropVault));
    }

    // Update the last claim timestamp
    lastClaim[msg.sender] = block.timestamp;

    // Update the claimed amount
    _claimedBy[msg.sender] += tokenAmountToDistribute;

    // Emit the TokenClaimed event
    emit TokenClaimed(msg.sender, tokenAmountToDistribute);

    // Transfer LoveTokens from the airdropVault to the user
    loveToken.transferFrom(address(airdropVault), msg.sender, tokenAmountToDistribute);
}
```
## <a id='H-03'></a>H-03. Sanity checks missing from `writeMessageInSharedSpace`, allowing unlimited messages and prev messages are overriten            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L106-L110


## Summary

The `Soulmate` contains potential vulnerabilities related to the handling of messages in the shared space. Two main issues have been identified: the lack of persistence for message history and the absence of limits on the number of messages a user can write. 

## Vulnerability Details
The current implementation overwrites the existing message in the shared space on each new request, resulting in a lack of message history.

Also, there is no restriction on the number of messages a user can write in the shared space, potentially leading to abuse and spam.

## Impact

1. **Lack of Message Persistence:**
   - *Severity:* Moderate
   - *Consequence:* Users may lose access to or reference previous messages, limiting the collaborative and interactive features of the shared space.

2. **Unlimited Message Writing:**
   - *Severity:* Moderate
   - *Consequence:* Malicious users could flood the shared space with an unlimited number of messages, impacting usability and readability.

## POC

- Run the below test via cmd `forge test --match-test testWriteMessageInSharedSpaceUnlimited -vvvv`

```
function testWriteMessageInSharedSpaceUnlimited() public {
    vm.prank(soulmate1);
    // can create unlimited messages, e.g change the `10` to `999999`
    for(uint256 i = 0; i < 10; ++i ) {
        soulmateContract.writeMessageInSharedSpace("Hello");
    }
    soulmateContract.writeMessageInSharedSpace("Hello last message over ritten");
    vm.prank(soulmate2);
    string memory message = soulmateContract.readMessageInSharedSpace();
    console2.log(soulmateContract.ownerToId(soulmate2));
    console2.log(message);
}
```
```
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 16.39ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommendations

1. **Message History Preservation:**
   - *Recommendation:* Modify the `writeMessageInSharedSpace` function to append new messages to the existing content in the shared space, allowing for the preservation of message history.
   - *Example:*
     ```solidity
     function writeMessageInSharedSpace(string calldata message) external {
         uint256 id = ownerToId[msg.sender];
         sharedSpace[id] = string.concat(sharedSpace[id], " ", message); // Append message instead of overwriting
         emit MessageWrittenInSharedSpace(id, message);
     }
     ```

2. **Limit on Message Writing:**
   - *Recommendation:* Implement limitations on the frequency or volume of messages a user can write in the shared space to prevent abuse and maintain a manageable shared space.
   - *Example:*
     ```solidity
     modifier limitMessageWriting() {
         // Implement logic to check and limit the frequency or volume of messages
         _;
     }

     function writeMessageInSharedSpace(string calldata message) external limitMessageWriting {
         uint256 id = ownerToId[msg.sender];
         sharedSpace[id] = string.concat(sharedSpace[id], " ", message);
         emit MessageWrittenInSharedSpace(id, message);
     }
     ```


## <a id='H-04'></a>H-04. Logical issue in `getDivorce` function, allows user to get divorce without having a soulmate            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L123-L129


## Summary

The `Soulmate` contains a potential vulnerability in the `getDivorced` function. Currently, any user, regardless of whether they have a soulmate or not, can call this function and trigger a divorce. This lack of access control could lead to unintended consequences and abuse. And logically it doesn't make sense that a user who don't have soulmate can get divorce.

## Vulnerability Details

The `getDivorced` function does not include a check to ensure that the caller has a soulmate before allowing the divorce operation to proceed. As a result, any user can call this function, potentially leading to unauthorized divorces and disrupting the intended functionality of the protocol.

```solidity
function getDivorced() public {
    address soulmate2 = soulmateOf[msg.sender];
    divorced[msg.sender] = true;
    divorced[soulmateOf[msg.sender]] = true;

    emit CoupleHasDivorced(msg.sender, soulmate2);
}
```

## Impact

The lack of access control in the `getDivorced` function poses several risks:

**Unauthorized Divorces**: Any user can call the function, even if they do not have a soulmate. This could lead to unauthorized divorces, disrupting the intended relationship management.

**Logical issue**: It doesn't make any sense that user who don't have a soulmate can get a divorce.

## POC

- Run the below test and it will pass successfully. Even the `soulmate1` has no soulmate, it can get divorce.

```solidity
function testGetDivorcedIfUserDontHaveSoulmate() public {
        vm.prank(soulmate1);
        soulmateContract.mintSoulmateToken();
        assertEq(soulmateContract.isDivorced(), false);

        soulmateContract.getDivorced();
        assertEq(soulmateContract.isDivorced(), true);

        vm.prank(soulmate2);
        console2.log(soulmateContract.isDivorced()); // false
        soulmateContract.mintSoulmateToken();
    }
```

Result:

```
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.98ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommendations

 **Access Control**: Implement access control mechanisms in the `getDivorced` function to ensure that only users with a soulmate can initiate a divorce.

   ```solidity
   modifier onlySoulmate(address account) {
       require(soulmateOf[account] != address(0), "Caller does not have a soulmate");
       _;
   }

   function getDivorced() public onlySoulmate(msg.sender) {
       address soulmate2 = soulmateOf[msg.sender];
       divorced[msg.sender] = true;
       divorced[soulmate2] = true;

       emit CoupleHasDivorced(msg.sender, soulmate2);
   }
   ```


## <a id='H-05'></a>H-05. initVault can be front-run, wrong `managerContract` can be deployed            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Vault.sol#L27-L31

## Summary
The `initVault` function in the `Vault` contract is declared as public, making it susceptible to front-running attacks. Additionally, there is a risk of deploying the wrong `managerContract` by any malicious actor. The `LoveToken` contract, when calling `initVault(address managerContract)`, approves a substantial amount to the specified `managerContract`, posing a potential security vulnerability.

## Vulnerability Details
The `initVault` function in the `Vault` contract is publicly accessible, which allows malicious actors to front-run the initialization, potentially causing unexpected behavior or exploiting the system.

There is a risk that a malicious actor deploys a different contract as the `managerContract` in the `LoveToken` contract, leading to unintended consequences during initialization.

If this happen then the `LoveToken` contract approves a significant amount of tokens to the `managerContract` without proper checks, creating a potential avenue for large-scale losses if the `managerContract` is compromised.

## Impact
1. **Front-running:** Malicious actors may exploit the public nature of `initVault` to manipulate the initialization process, leading to undesirable outcomes.
2. **Incorrect `managerContract`:** Deploying the wrong `managerContract` can result in unexpected behaviors, potentially compromising the security of the system.
3. **Excessive Token Approval:** Approving a large token amount without adequate checks may lead to substantial losses if the `managerContract` is compromised.

## Tools Used
Manual code review and analysis.

## Recommendations

When deploying the contracts make sure to deploy and initialize in same transaction to avoid the front-running. 
		


# Low Risk Findings

## <a id='L-01'></a>L-01. Deposit and withdrawals validation checks missing            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L50-L66


## Summary

The `Staking` contains issues in the `deposit` and `withdraw` functions where the comments suggest an increase or decrease in the `userStakes` variable, respectively. However, there is no validation for a zero amount deposit or withdrawal, which contradicts the logical comments. Additionally, the `withdraw` function lacks a check for whether the user has deposited before attempting to decrease the `userStakes` variable. 

## Vulnerability Details
The `deposit` function comment suggests increasing the `userStakes` variable, but it allows for zero amount deposits without any validation.

The `withdraw` function comment suggests decreasing the `userStakes` variable, but it allows for zero amount withdrawals without any validation.

The `withdraw` function also lacks a check to determine whether the user has deposited before attempting to decrease the `userStakes` variable.

## Impact

1. **Zero Amount Deposit:**
   - *Severity:* Low
   - *Consequence:* Allowing zero amount deposits could lead to unexpected behavior and may contradict the intended logic of the `deposit` function.

2. **Zero Amount Withdrawal:**
   - *Severity:* Low
   - *Consequence:* Permitting zero amount withdrawals may result in unexpected outcomes and could conflict with the intended logic of the `withdraw` function.

3. **Missing User Deposit Check:**
   - *Severity:* Moderate
   - *Consequence:* The absence of a check to verify whether the user has deposited before withdrawal may lead to inconsistencies in the `userStakes` variable and unintended consequences.

## POC

- Copy below test and run it via cmd `forge test --match-test testUserDepositAndWithdrawlWithZeroAmount -vvvv`

```
function testUserDepositAndWithdrawlWithZeroAmount() public {
        uint amount = 0 ether;
        loveToken.approve(address(stakingContract), amount);
        console2.log("Before Deposit:", loveToken.balanceOf(address(stakingContract)));
        stakingContract.deposit(amount);
        console2.log("After Deposit:", loveToken.balanceOf(address(stakingContract)));

        vm.prank(soulmate1);
        console2.log("Balance of user after Withdrawal:", loveToken.balanceOf(soulmate1));
        stakingContract.withdraw(amount);
        console2.log("Balance of user after Withdrawal:", loveToken.balanceOf(soulmate1));
}
```

Result:
```
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.03ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommendations

**Deposit Function:**
   - *Recommendation:* Add a validation check to ensure that the `amount` is greater than zero before updating the `userStakes` variable in the `deposit` function.
   - *Example:*
     ```solidity
     require(amount > 0, "Amount must be greater than zero");
     userStakes[msg.sender] += amount;
     ```

**Withdraw Function:**
   - *Recommendation 1:* Add a validation check to ensure that the `amount` is greater than zero before updating the `userStakes` variable in the `withdraw` function.
     - *Example:*
       ```solidity
       require(amount > 0, "Amount must be greater than zero");
       userStakes[msg.sender] -= amount;
       ```

   - *Recommendation 2:* Add a check to verify whether the user has deposited before proceeding with the withdrawal operation.
     - *Example:*
       ```solidity
       require(userStakes[msg.sender] >= amount, "Insufficient funds");
       userStakes[msg.sender] -= amount;
       ```


## <a id='L-02'></a>L-02. Wrong data emitted in events of `LoveToken`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/LoveToken.sol#L46-L56

## Summary
The `initVault` function in the `LoveToken` contract emits both the `AirdropInitialized` and `StakingInitialized` events with incorrect data. The event data indicates that the parameters passed should be `airdropContract` and `stakingContract`, respectively. Still, the function emits both events with the parameter `managerContract`, resulting in inaccurate event logs.

## Vulnerability Details
Both the `AirdropInitialized` and `StakingInitialized` events are expected to log the addresses of the corresponding contracts (`airdropContract` and `stakingContract`). However, the `initVault` function incorrectly emits both events with the parameter `managerContract`, leading to a discrepancy between the event's data and the actual initialized contracts.

## Impact
The incorrect event data poses a challenge for developers and external systems relying on event logs, as the information provided does not accurately reflect the initialized contracts. This discrepancy may result in confusion and hinder proper tracking of contract initialization events.

## Tools Used
Manual review.

## Recommendations
 **Correct Event Data:** Update the `initVault` function to emit both the `AirdropInitialized` and `StakingInitialized` events with the correct parameters, reflecting the addresses of the corresponding contracts (`airdropContract` and `stakingContract`).



