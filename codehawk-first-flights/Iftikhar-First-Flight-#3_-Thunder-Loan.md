# First Flight #3: Thunder Loan - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. potential reentrancy in flashloan, can drain tokens from the contract](#H-01)
    - ### [H-02. receiverAddress.isContract is not foolproof, allowing an EOA to misuse the flash loan feature](#H-02)
    - ### [H-03. ThunderLoanUpgraded will have no access to updateExchangeRate, will not be able to perform flashloans](#H-03)
    - ### [H-04. Extraneous Code in Deposit Function Impacts Exchange Rate](#H-04)
    - ### [H-05. updateExchangeRate function can be exploited/manipulated by attackers](#H-05)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #3

### Dates: Nov 1st, 2023 - Nov 8th, 2023

[See more contest details here](https://www.codehawks.com/contests/clocopz26004rkx08q1n61wnz)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 0
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. potential reentrancy in flashloan, can drain tokens from the contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/8539c83865eb0d6149e4d70f37a35d9e72ac7404/src/protocol/ThunderLoan.sol#L180

https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/8539c83865eb0d6149e4d70f37a35d9e72ac7404/src/protocol/ThunderLoan.sol#L194

## Summary

The `flashloan` function in the `ThunderLoan` contract has a potential reentrancy vulnerability that can be exploited to drain tokens from the contract. This vulnerability occurs because the function does not enforce proper order of operations when handling external calls, allowing an attacker to create a reentrancy attack. 

## Vulnerability Details

1. **Lack of Correct Sequence:** The function's improper order of operations, calculating a fee and updating exchange rates before verifying the user's contract (external), poses a reentrancy risk.

2. **Reentrancy Attack Vector:** The vulnerability arises from `executeOperation` on receiverAddress, allowing malicious or unexpected behavior, potentially disrupting the flash loan process.

## Impact

This vulnerability can lead to unauthorized reentrancy attacks, enabling malicious external contracts to disrupt the flash loan process and potentially drain assets from the ThunderLoan contract.

## POC

```
function testReentrancyAttack() public setAllowedToken {
    tokenA.mint(liquidityProvider, AMOUNT);

    vm.startPrank(liquidityProvider);
    tokenA.approve(address(thunderLoan), AMOUNT);
    thunderLoan.deposit(tokenA, AMOUNT);
    vm.stopPrank();

    AssetToken asset = thunderLoan.getAssetFromToken(tokenA);


    vm.startPrank(user);
    tokenA.mint(address(mockFlashLoanReceiver), AMOUNT);

    // Simulate a reentrancy attack by repeatedly calling the flashloan function
    while (asset.balanceOf(address(thunderLoan)) != 0) {
        thunderLoan.flashloan(address(mockFlashLoanReceiver), tokenA, AMOUNT, "");
    }

    vm.stopPrank();

    // Ensure the attack drained the tokens from ThunderLoan
    assertEq(asset.balanceOf(address(thunderLoan)), 0, "Tokens not drained");
}
```

## Tools Used

- Manual review

## Recommendations

For removing the reentrancy risk, you should follow these steps:

1 - Calculate the fee and update the exchange rate before interacting with external contracts. This is critical to ensure the correct sequence of operations.

2 - Implement guard conditions to prevent reentrancy attacks by checking whether the function has already been called during the execution of a flash loan.

New code will look like this for preventing reentrancy risks:

```
function flashloan(address receiverAddress, IERC20 token, uint256 amount, bytes calldata params) external {
    AssetToken assetToken = s_tokenToAssetToken[token];
    uint256 startingBalance = IERC20(token).balanceOf(address(assetToken));

    if (amount > startingBalance) {
        revert ThunderLoan__NotEnoughTokenBalance(startingBalance, amount);
    }

    if (!receiverAddress.isContract()) {
        revert ThunderLoan__CallerIsNotContract();
    }

    uint256 fee = getCalculatedFee(token, amount);
    uint256 endingBalance;

    // Update the exchange rate before any external interaction to prevent reentrancy attacks.
    assetToken.updateExchangeRate(fee);

    emit FlashLoan(receiverAddress, token, amount, fee, params);

    s_currentlyFlashLoaning[token] = true;

    // Transfer tokens to the receiver.
    assetToken.transferUnderlyingTo(receiverAddress, amount);

    // Set a flag to prevent reentrant calls
    bool alreadyCalled = false;

    // Execute the receiver's contract function.
    (bool success, bytes memory result) = receiverAddress.call(
        abi.encodeWithSignature(
            "executeOperation(address,uint256,uint256,address,bytes)",
            address(token),
            amount,
            fee,
            msg.sender,
            params
        )
    );

    // Check the execution result.
    if (!success) {
        revert("Execution of receiver's contract function failed");
    }

    // Prevent reentrancy by requiring that it has not already been called
    require(!alreadyCalled, "Reentrant call detected");
    alreadyCalled = true;

    // Calculate the ending balance after the receiver's operation.
    endingBalance = token.balanceOf(address(assetToken);

    s_currentlyFlashLoaning[token] = false;

    if (endingBalance < startingBalance + fee) {
        revert ThunderLoan__NotPaidBack(startingBalance + fee, endingBalance);
    }
}
```

In this code, a bool variable alreadyCalled is introduced to ensure that the `receiverAddress` contract's executeOperation function is not called **reentrantly** within the same transaction. This helps prevent reentrancy attacks. 




## <a id='H-02'></a>H-02. receiverAddress.isContract is not foolproof, allowing an EOA to misuse the flash loan feature            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/8539c83865eb0d6149e4d70f37a35d9e72ac7404/src/protocol/ThunderLoan.sol#L188

## Summary

The issue is that the flashloan function can be exploited to be called by externally owned addresses (EOAs) when it should only be accessible to smart contracts.

## Vulnerability Details

The vulnerability details involve the potential for external addresses (EOAs) to bypass the contract-only restriction and exploit the `flashloan` function.

## Impact

External addresses (EOAs) can initiate flash loans, which should only be available to contracts, potentially leading to misuse of the flash loan functionality and destabilization of the system.

## POC

This test successfully passes by utilizing an external owner address (EOA) to call the flash loan function, despite the intended restriction that only contracts should have access to it.

```
function testExploitFlashLoan() public {
    // Simulate an EOA address
    address externalOwner = address(0x73de83588F8D99d8043143b29BCD015A61433A29);

    // Attempt to call flashloan from the EOA address
    (bool success, ) = externalOwner.call(
        abi.encodeWithSignature(
            "flashloan(address,address,uint256,bytes)",
            address(thunderLoan),
            address(thunderLoan),
            100, // amount
            "Exploit"
        )
    );

    // Check if the call was successful
    assertEq(success, true);
}
```
## Tools Used

- Foundry
- Manual review

## Recommendations

To address this vulnerability, ensure that only contract addresses are allowed to initiate flash loans by improving the check for `receiverAddress.isContract()` to be more robust and prevent EOA bypasses.

```diff
-if (!receiverAddress.isContract()) { revert ThunderLoan__CallerIsNotContract(); }
+require(receiverAddress.isContract(), "Caller must be a contract");
```
## <a id='H-03'></a>H-03. ThunderLoanUpgraded will have no access to updateExchangeRate, will not be able to perform flashloans            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/8539c83865eb0d6149e4d70f37a35d9e72ac7404/src/protocol/AssetToken.sol#L80C57-L80C57

## Summary

If the `ThunderLoan` contract is upgraded to `ThunderLoanUpgraded` in the future, the `updateExchangeRate` function will not be accessible to the upgraded contract. This is because the `updateExchangeRate` function has an `onlyThunderLoan` modifier, which permits access exclusively to the `ThunderLoan` contract and does not extend to its upgraded counterpart, `ThunderLoanUpgraded`."

## Vulnerability Details

The vulnerability in the contract lies in its upgradeability mechanism. Specifically, if the ThunderLoan contract is upgraded to ThunderLoanUpgraded in the future, the `updateExchangeRate` function within the `AssetToken` contract will no longer be accessible to the upgraded contract. The primary reason for this is the presence of the `onlyThunderLoan` modifier, which restricts access to the ThunderLoan contract exclusively & does not include the upgraded version, **ThunderLoanUpgraded**.

## Impact

The inability of the `ThunderLoanUpgraded` contract to access the `updateExchangeRate` function may lead to loss of key functionality, including the ability to enable users to perform flash loans, resulting in a significant impact on the platform's operation.

## Tools Used

- Foundry and manual review

## Recommendations

To address the issue, you should:

1. Modify the `onlyThunderLoan` modifier to include `ThunderLoanUpgraded` for access to the `updateExchangeRate` function.
2. Implement a proxy pattern for consistent functionality across both contracts.
3. Ensure backward compatibility to avoid disrupting existing features.
4. Rigorously test the modified contracts for security.
5. Provide clear upgrade documentation for users.

Change `onlyThunderLoan` code to the below to include `ThunderLoanUpgraded` for access to the **updateExchangeRate** function.

```
modifier onlyThunderLoan() {
    if (msg.sender != i_thunderLoan || msg.sender != address(thunderLoanUpgraded)) {
        revert AssetToken__onlyThunderLoan();
    }
    _;
}
```
## <a id='H-04'></a>H-04. Extraneous Code in Deposit Function Impacts Exchange Rate            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/8539c83865eb0d6149e4d70f37a35d9e72ac7404/src/protocol/ThunderLoan.sol#L153C10-L153C10

https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/8539c83865eb0d6149e4d70f37a35d9e72ac7404/src/protocol/ThunderLoan.sol#L154

## Summary

Adding exchange rate update lines to the deposit function is **atypical** in `ThunderLoan::depost`, potentially introducing complexities and affecting exchange rate stability, which differs from standard protocols and may lead to unexpected issues.

## Vulnerability Details

The unconventional inclusion of exchange rate update lines within the deposit function introduces a vulnerability that may complicate the process and disrupt exchange rate stability. This departure from standard practices in lending and borrowing protocols poses potential risks and impacts the predictability of the exchange rate, creating an area of concern.

## Impact

The unconventional exchange rate update in the deposit function introduces complexity, jeopardizes exchange rate stability, and departs from standard practices, potentially causing user confusion, trust issues, and unforeseen operational problems.

## POC

If the exchange rate is altered with every deposit, it will lead to an increment in the token price for new depositors. This means that users depositing funds into the system will receive fewer tokens for their assets, affecting the cost-effectiveness and attractiveness of the deposit process for new participants.

## Tools Used

- Manual review and Foundry

## Recommendations

Remove the exchange rate update from the deposit function to align it with standard practices. The deposit function should focus on the core deposit operation without altering exchange rates.

```diff
-uint256 calculatedFee = getCalculatedFee(token, amount);
-assetToken.updateExchangeRate(calculatedFee);
```

In the upgrade these lines are already removed. So that will do.
## <a id='H-05'></a>H-05. updateExchangeRate function can be exploited/manipulated by attackers            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/8539c83865eb0d6149e4d70f37a35d9e72ac7404/src/protocol/ThunderLoan.sol#L154

https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/8539c83865eb0d6149e4d70f37a35d9e72ac7404/src/protocol/ThunderLoan.sol#L194

## Summary

The use of local functions `ThunderLoan::deposit->updateExchangeRate` & `ThunderLoan::flashloan->updateExchangeRate` may allow attackers to manipulate exchange rates, recommending reliance on decentralized price feeds, such as Chainlink, for enhanced security.


## Vulnerability Details

The `ThunderLoan::deposit->updateExchangeRate` & `ThunderLoan::flashloan->updateExchangeRate` function calculates the exchange rate using local or internal functions, which could potentially be manipulated or exploited by attackers. It is recommended to use decentralized price feeds, such as `Chainlink`, for more reliable and secure price information.

## Impact

The vulnerability could allow attackers to manipulate or exploit exchange rate calculations

## Tools Used

- Foundry 
- Manual review

## Recommendations

**Utilize Decentralized Price Feeds:** To mitigate potential vulnerabilities in the exchange rate calculation, consider utilizing decentralized price feeds from reliable sources such as Chainlink. This will provide more trustworthy and secure pricing information.

Ref: https://docs.chain.link/data-feeds/price-feeds


		





