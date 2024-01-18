# First Flight #7: Horse Store - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Lack of withdraw function in `payable` mint macro, funds can stuck in contract forever](#H-01)
    - ### [H-02. Incorrect evaluation in `IS_HAPPY_HORSE` macro](#H-02)
    - ### [H-03. Non-Payable Entry Point in MAIN Macro of HUFF contract](#H-03)
    - ### [H-04. Horses must be able to be fed at all times invariant can be breaked](#H-04)
    - ### [H-05.  Absence of Horse Existence Check in `FEED_HORSE` Macro Allows Feeding Without Minting a Horse](#H-05)
    - ### [H-06. User can't mint more than one horse in huff](#H-06)
    - ### [H-07. Unchecked horse existence in `feedHorse` function](#H-07)
- ## Medium Risk Findings
    - ### [M-01. Unprotected function dispatch in `MAIN` Macro](#M-01)
    - ### [M-02. Compiler Version Incompatibility with Layer 2 Networks: Deployment Issues on Arbitrum](#M-02)
- ## Low Risk Findings
    - ### [L-01. Unused macro `GET_HORSE_FED_TIMESTAMP`](#L-01)
    - ### [L-02. Unused Macro `HORSE_HAPPY_IF_FED_WITHIN()` in HorseStore Huff Code](#L-02)
    - ### [L-03. Input Parameter Mismatch in `feedHorse` & `isHappyHorse` functions](#L-03)
    - ### [L-04. `IS_HAPPY_HORSE` wrong returns value in definition](#L-04)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #7

### Dates: Jan 11th, 2024 - Jan 18th, 2024

[See more contest details here](https://www.codehawks.com/contests/clr6s75ut00013qg9z8bpkalo)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 7
   - Medium: 2
   - Low: 4


# High Risk Findings

## <a id='H-01'></a>H-01. Lack of withdraw function in `payable` mint macro, funds can stuck in contract forever            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L318-L352

## Summary
There is a potential vulnerability in the Mint function of the `HorseStore.huff`, denoted by the macro _MINT(). The issue arises from the comment indicating that the Mint function is `payable` without a corresponding withdraw function, leading to a situation where Ether (ETH) sent to the contract can become stuck indefinitely. 

## Vulnerability Details
The vulnerability is associated with the Mint function, as defined by the macro _MINT(). Specifically, the function lacks a mechanism to withdraw Ether (ETH) from the contract after it has been received. Without a withdraw function, any Ether sent to the contract through the Mint function will remain trapped within the contract, creating a potential loss of funds for users.

```solidity
/// @notice Mint
/// @notice Mints a new token
/// @dev The Mint function is payable
#define macro _MINT() = takes (2) returns (0) {
    // ... (function implementation)
}
```

## Impact
The absence of a withdraw function in the Mint operation means that any Ether sent to the contract through this function cannot be retrieved. As a result, users who mint tokens and send Ether along with the transaction may face a loss of funds, as there is no mechanism to extract the trapped Ether from the contract.

## Tools Used
Manual review.

## Recommendations
To address the identified vulnerability, the following recommendations are provided:

 **Add a Withdraw Function**: Implement a withdraw function that allows the contract owner or users to retrieve any Ether stuck in the contract. 

OR another alternative is:

**Make Mint Function Non-Payable**: Consider making the Mint function non-payable if there is no intended functionality requiring Ether to be sent along with token minting. This can prevent users from inadvertently sending Ether to the contract and facing potential loss.

## <a id='H-02'></a>H-02. Incorrect evaluation in `IS_HAPPY_HORSE` macro            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L93-L115

## Summary

There is an issue in the inline assembly code within the `IS_HAPPY_HORSE` macro. The current code fails to correctly handle the case where `timestamp` is equal to `horseFedTimestamp`, resulting in an incorrect return value.

## Vulnerability Details

The issue arises in the following portion of the code:

```assembly
eq                                  // [timestamp == horseFedTimestamp]
start_return_true jumpi             // [timestamp, horseFedTimestamp]
```

The code incorrectly jumps to the start_return_true label only if `timestamp` is not equal to `horseFedTimestamp`, leading to an incorrect result when they are equal.

## Impact

This vulnerability could lead to incorrect evaluation of whether a horse is happy or not. In particular, when `timestamp` matches `horseFedTimestamp`, the current code incorrectly returns false, contrary to the expected behavior.

## Tools Used

Manual review.

## Recommendations

It is recommended to modify the code to handle the case where `timestamp` is equal to `horseFedTimestamp` correctly. 

Update code will look like this:

```huff
#define macro IS_HAPPY_HORSE() = takes (0) returns (0) {
    0x04 calldataload                   // [horseId]
    LOAD_ELEMENT(0x00)                  // [horseFedTimestamp]
    timestamp                           // [timestamp, horseFedTimestamp]
    dup2 dup2                           // [timestamp, horseFedTimestamp, timestamp, horseFedTimestamp]
    sub                                 // [timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
    [HORSE_HAPPY_IF_FED_WITHIN_CONST]   // [HORSE_HAPPY_IF_FED_WITHIN, timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
    dup2                                // [HORSE_HAPPY_IF_FED_WITHIN, timestamp - horseFedTimestamp, timestamp, horseFedTimestamp, timestamp - horseFedTimestamp]
    eq                                  // [HORSE_HAPPY_IF_FED_WITHIN == timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
    start_return_true jumpi             // [timestamp, horseFedTimestamp]
    eq                                  // [timestamp == horseFedTimestamp, timestamp, horseFedTimestamp]
    start_return_true jumpi             // [timestamp, horseFedTimestamp]
    0x00                                // [0, timestamp, horseFedTimestamp]
    start_return_true:

    start_return:
    // Store value in memory.
    0x00 mstore

    // Return value
    0x20 0x00 return
}
```

## <a id='H-03'></a>H-03. Non-Payable Entry Point in MAIN Macro of HUFF contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L144-L206

## Summary

The `MAIN()` macro in the `HorseStore.huff`, which serves as the entry point for various functions, is by default payable. However, as there is no functionality to withdraw Ether from the contract, the contract should not accept Ether. 

## Vulnerability Details

The vulnerability lies in the original implementation of the `MAIN()` macro, which allowed the contract to receive Ether despite not having any functionality to handle or withdraw it. This could potentially result in unintended Ether accumulation within the contract.

## Impact

There is no withdrawal mechanism for Ether in the contract. So ETH can get stuck in contract forever.

## Tools Used

Manual review.

## Recommendations

It is recommended to explicitly handle Ether transactions in a manner appropriate for the contract's intended functionality. Since there is no Ether withdrawal mechanism, making the `MAIN()` macro non-payable will be a better choice. 

```huff
#define macro MAIN() = takes (0) returns (0) {
    // Check if Ether is sent, and revert if true
    callvalue throw_error jumpi

    // Identify which function is being called.
    0x00 calldataload 0xE0 shr // this pushes the signature onto the stack (1st 4 bytes)

    // Rest of the existing logic...

    throw_error:
        0x00 0x00 revert
}
```
## <a id='H-04'></a>H-04. Horses must be able to be fed at all times invariant can be breaked            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L80-L91

## Summary

The HorseStore.huff contains a critical issue that violates a strict protocol invariant. According to the protocol, "Horses must be able to be fed at all times." However, a condition in the code (`0x11 timestamp mod`) causes the feeding transaction to revert if the time difference between the current block timestamp and the last fed timestamp is a multiple of 17.

## Vulnerability Details

In the `feedHorse` function, the code includes a check `(block.timestamp - horseIdToFedTimeStamp[horseId]) % 17 == 0` before allowing the horse to be fed. This condition leads to a revert if the time difference is a multiple of 17. As a result, the invariant stating that "Horses must be able to be fed at all times" is violated.

### Code snippet:
```huff
#define macro FEED_HORSE() = takes (0) returns (0) { 
    timestamp               // [timestamp]
    0x04 calldataload       // [horseId, timestamp]
    STORE_ELEMENT(0x00)     // []

    // End execution 
    0x11 timestamp mod      
    endFeed jumpi                
    revert 
    endFeed:
    stop
}
```
## Impact

The impact of this issue is significant, as it directly contradicts a fundamental protocol requirement. Users may face unexpected transaction reversals when attempting to feed horses, leading to disruptions in the expected behavior of the smart contract. This could result in confusion and financial losses for users relying on the correct functioning of the feeding mechanism.

## Tools Used

Manual code review.

## Recommendations

Remove the unnecessary code:

```diff
-0x11 timestamp mod      
-endFeed jumpi                
-revert 
-endFeed:
-stop
```

## <a id='H-05'></a>H-05.  Absence of Horse Existence Check in `FEED_HORSE` Macro Allows Feeding Without Minting a Horse            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L80-L91


## Summary
In `HorseStore.huff` in the `FEED_HORSE` macro, there is an issue revolves around the absence of a check for the existence of a horse before allowing it to be fed.

## Vulnerability Details

The `FEED_HORSE` macro does not include a check to verify the existence of the horse with the given `horseId`. This omission allows feeding a horse without ensuring that it has been minted previously.

### Code Snippet
```huff
#define macro FEED_HORSE() = takes (0) returns (0) {
    // Existing logic...

    // End execution 
    0x11 timestamp mod                  
    endFeed jumpi                        
    revert 
    endFeed:
    stop
}
```

## Impact

It allows feeding a horse without ensuring its prior minting. This could lead to inconsistencies in the contract state and potential unexpected behavior. Basically, this break the invariant because how can horse be happy if it is not present?

## POC
- Copy the below code
- Run it via `forge test --match-test testStableMasterIsFeedingToGhostsInsteadOfHorsesInHuff`

```solidity
function testStableMasterIsFeedingToGhostsInsteadOfHorsesInHuff() public {
    uint256 horseId = horseStore.totalSupply();
    vm.warp(horseStore.HORSE_HAPPY_IF_FED_WITHIN());
    vm.roll(horseStore.HORSE_HAPPY_IF_FED_WITHIN());
    vm.prank(user);
    horseStore.feedHorse(horseId);
    assertEq(horseStore.isHappyHorse(horseId), true);
}
```

Results:

```
Running 1 test for test/HorseStoreHuff.t.sol:HorseStoreHuff
[PASS] testStableMasterIsFeedingToGhostsInsteadOfHorsesInHuff() (gas: 37993)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.79s
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual review.

## Recommendation
Include a check to verify the existence of the horse with the given `horseId` before proceeding with the feeding logic. This will prevent feeding a horse that has not been minted.

### New code should be something like this:
```huff
#define macro FEED_HORSE() = takes (0) returns (0) {
    // Load horseId from calldata
    0x04 calldataload                   // [horseId]

    // Check if the horse with the given horseId exists
    [OWNER_LOCATION] LOAD_ELEMENT_FROM_KEYS(0x00)   // [owner, horseId]
    invalid_horse jumpi                // [horseId]

    // Continue with feeding logic
    timestamp                           // [timestamp]
    STORE_ELEMENT(0x00)                 // []

    // End execution 
    0x11 timestamp mod                  
    endFeed jumpi                        
    revert 
    endFeed:
    stop

    invalid_horse:
        INVALID_HORSE(0x00)
}
```

## <a id='H-06'></a>H-06. User can't mint more than one horse in huff            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L73-L78


## Summary

The `MINT_HORSE` macro in HuffStore.huff contract has a vulnerability that restricts users from minting more than one horse. The macro or associated `_MINT` function contains logic preventing unlimited minting, contrary to the intended behavior.

## Vulnerability Details

The `MINT_HORSE` macro in HuffStore.huff contains logic that restricts users to minting only one horse. This limitation is not aligned with the project requirements, where users should be allowed to mint an unlimited number of horses.

**Code snippet:**
```solidity
#define macro MINT_HORSE() = takes (0) returns (0) {
    [TOTAL_SUPPLY] // [TOTAL_SUPPLY]
    caller         // [msg.sender, TOTAL_SUPPLY]
    _MINT()        // [] 
    stop           // []
}
```

## Impact

The impact of this issue is significant as it directly contradicts the project requirements. Users are unable to mint more than one horse, leading to a deviation from the expected behavior. This limitation may affect user experience and hinder the project's functionality.

## POC

- Copy the below test
- Run it via `forge test --match-test testUserCannotMintUnlimitedInHuff -vvv` and you will see the below result

```solidity
function testUserCannotMintUnlimitedInHuff() public {
    vm.prank(user);
    horseStore.mintHorse();
    vm.expectRevert();
    horseStore.mintHorse();
}
```

Result:
```
Running 1 test for test/HorseStoreHuff.t.sol:HorseStoreHuff
[PASS] testUserCannotMintUnlimitedInHuff() (gas: 60319)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.93s
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Also, digging more into this. It seems like the total supply is NOT increasing. Run the below test for proof;

```solidity
function testTotalSupplyIsNotIncreasing() public {
    uint256 beforeTotalSupply = horseStore.totalSupply();
    vm.prank(user);
    horseStore.mintHorse();
    assertNotEq(horseStore.totalSupply(), beforeTotalSupply + 1);
}
```

## Tools Used

Manual review.

## Recommendations

1. **Update Minting Logic:**
   - Revise the logic within the `_MINT` function or the `MINT_HORSE` macro to allow users to mint an unlimited number of horses.

2. **Verify TOTAL_SUPPLY Handling:**
   - Ensure that the handling of `TOTAL_SUPPLY` is appropriately implemented and does not unintentionally restrict minting.


## <a id='H-07'></a>H-07. Unchecked horse existence in `feedHorse` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.sol#L32-L34


## Summary

The `feedHorse` function in the `HorseStore` contract allows updating the `horseIdToFedTimeStamp[horseId]` for a given `horseId` without verifying whether the horse with that `horseId` has been minted. This oversight can lead to a violation of the specified invariant and inaccurate happiness checks in the `isHappyHorse` function.

## Vulnerability Details

The issue arises in the `feedHorse` function, where the `horseId` is not checked for existence (minting) before updating the associated `horseIdToFedTimeStamp[horseId]`. This oversight can result in misleading information, wasted gas, and a violation of the invariant.

## Impact

1. **Breaking Invariant:**
   - This break the invariant because how can horse be happy if it is not present? 

2. **Misleading Information:**
   - Feeding non-existent horses can result in inaccurate information about the state of horses, as `horseIdToFedTimeStamp[horseId]` may be updated for non-existent horses.

3. **Inconsistent State:**
   - The contract may end up in an inconsistent state with certain `horseId` values having associated timestamps but no corresponding horses.

## POC

- Copy the below test code
- Run it via this cmd `forge test --match-test testStableMasterIsFeedingToGhostsInsteadOfHorses`

Test code:

```solidity
function testStableMasterIsFeedingToGhostsInsteadOfHorses() public {
    uint256 horseId = horseStore.totalSupply();
    vm.warp(horseStore.HORSE_HAPPY_IF_FED_WITHIN());
    vm.roll(horseStore.HORSE_HAPPY_IF_FED_WITHIN());
    vm.prank(user);
    horseStore.feedHorse(horseId);
    assertEq(horseStore.isHappyHorse(horseId), true);
}
```

It will result in this:

```
Running 1 test for test/HorseStoreSolidity.t.sol:HorseStoreSolidity
[PASS] testStableMasterIsFeedingToGhostsInsteadOfHorses() (gas: 38966)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.02ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual review.

## Recommendations

**Check Horse Existence:**
   - Add a check in the `feedHorse` function to ensure that the provided `horseId` corresponds to a valid horse (exists) before updating the mapping.

e.g you can add a check like this to avoid this issue:

```
// Check if the horse with the given horseId exists before updating the timestamp
require(exists(horseId), "Horse does not exist");
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Unprotected function dispatch in `MAIN` Macro            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L144-L206

## Summary

The `MAIN` macro in the `HorseStore.huff` contract lacks a revert instruction in case none of the specified function signatures match the provided calldata. This absence of a revert mechanism may lead to unintended consequences, such as calling the wrong function, potentially causing security vulnerabilities or unintended state changes.

## Vulnerability Details

The `MAIN` macro is designed to identify and execute the appropriate function based on the provided calldata. However, it does not include a revert instruction at the end to handle the scenario where none of the specified function signatures match the input. As a result, if a caller provides calldata with an unrecognized signature, the contract will proceed to execute the first function specified in the `MAIN` macro (`GET_TOTAL_SUPPLY` in this case). 

## Impact

The absence of a revert mechanism in the `MAIN` macro could potentially lead to the execution of unintended functions. In a scenario where sensitive functions or state-changing operations are defined later in the macro, this could result in unexpected behavior or security vulnerabilities. It may allow malicious actors to trigger undesired actions within the contract.

## Tools Used

Manual review.

## Recommendations

It is recommended to add a revert instruction in the`MAIN` macro to handle the case where none of the specified function signatures match the provided calldata. This will ensure that the contract reverts in case of unrecognized function signatures, preventing unintended execution and enhancing the overall security of the contract.

```huff
0x00 0x00 revert
```
## <a id='M-02'></a>M-02. Compiler Version Incompatibility with Layer 2 Networks: Deployment Issues on Arbitrum            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.sol#L2

## Summary

The protocol employs Solidity compiler version 0.8.20, yet it aims for deployment on Arbitrum, a Layer 2 (L2) network. However, this introduces bytecode complications involving the `PUSH0` opcode, which is incompatible with Arbitrum. The recommended solution is to either revert to Solidity compiler version 0.8.19 or incorporate `evm_version = "paris"` in the `foundry.toml` configuration file to ensure compatibility with Arbitrum.

## Vulnerability Details

The issue lies in the bytecode generated by Solidity compiler version 0.8.20, specifically with the `PUSH0` opcode, causing deployment failures on L2 networks like Arbitrum. This arises due to the lack of support for PUSH0 on certain L2 networks.

## Impact

Deployment failures on Arbitrum or other L2 networks pose a significant hindrance to the protocol's successful launch. The incompatibility may affect the functionality and accessibility of the protocol on these networks.

## Tools Used

Manual review.

## Recommendations

To address the problem, the recommended course of action is to downgrade the Solidity compiler version to 0.8.19 or insert `evm_version = "paris"` in the `foundry.toml` configuration file. This ensures alignment with L2 networks like Arbitrum, avoiding deployment failures and ensuring the protocol's compatibility.



# Low Risk Findings

## <a id='L-01'></a>L-01. Unused macro `GET_HORSE_FED_TIMESTAMP`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L64-L71

## Summary

The macros, namely `GET_HORSE_FED_TIMESTAMP()`, is defined but not referenced or utilized anywhere in the contract code.

## Vulnerability Details

The identified issues revolve around the existence of unused macro within the codebase.  GET_HORSE_FED_TIMESTAMP(), is designed to provide functionality that is not utilized in the contract's logic. As stated in the documentation, the goal is to optimize the contract in Huff, and maintaining unused or unnecessary code can lead to confusion and inefficiency.


## Impact

The impact of these issue is primarily related to code clarity, maintenance, and gas consumption. Unused code can contribute to confusion among developers and may hinder future updates or modifications to the contract. Additionally, deploying and executing unnecessary code can result in increased gas costs without providing any meaningful functionality.

## Tools Used

Manual review

## Recommendations

The recommended action is to remove the unused macros `HORSE_HAPPY_IF_FED_WITHIN()` cuz it's not used anywhere in the code.
## <a id='L-02'></a>L-02. Unused Macro `HORSE_HAPPY_IF_FED_WITHIN()` in HorseStore Huff Code            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L58-L62

## Summary

The macro, named `HORSE_HAPPY_IF_FED_WITHIN()`, is defined to return a constant value, but it is not referenced or utilized anywhere in the contract code. This raises concerns about code cleanliness, potential confusion, and unnecessary gas consumption.

## Vulnerability Details

The identified issue revolves around the existence of an unused macro within the codebase. The macro is designed to return a constant value but has no practical application within the contract's logic. As stated in the documentation, the goal is to optimize the contract in Huff, and maintaining unused or unnecessary code can lead to confusion and inefficiency.

## Impact

The impact of this issue is primarily related to code clarity, maintenance, and gas consumption. Unused code can contribute to confusion among developers and may hinder future updates or modifications to the contract. Additionally, deploying and executing unnecessary code can result in increased gas costs without providing any meaningful functionality.

## Tools Used

Manual review.

## Recommendations

The recommended action is to remove the unused macro `HORSE_HAPPY_IF_FED_WITHIN()`. Since it does not contribute to the functionality of the contract, its presence introduces unnecessary complexity and potential confusion for developers.

## <a id='L-03'></a>L-03. Input Parameter Mismatch in `feedHorse` & `isHappyHorse` functions            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L80-L115


## Summary
The `feedHorse` and `isHappyHorse` functions in the `HorseStore` huff code lack specification of their input parameters in their corresponding macros. This oversight can lead to confusion and errors during code execution, as it may not be clear to developers or auditors what input parameters are expected by these functions.

## Vulnerability Details
The vulnerability arises from the discrepancy between the function declarations and their corresponding macro implementations. In the function declarations, both `feedHorse` and `isHappyHorse` specify a `uint256` input parameter, indicating that they expect a `horseId` as input. However, in their macro implementations, the input parameters are not explicitly defined, leading to potential issues in understanding and using these macros.

## Impact
The impact of this issue is primarily on code clarity and maintainability. Developers and auditors reviewing the code may find it challenging to understand the intended usage of these macros due to the absence of explicit input parameter specifications. This can result in misinterpretation, potential errors during integration, and difficulties in maintaining or modifying the code.

## Tools Used
Manual review.

## Recommendations
To address this issue, it is recommended to update the macro definitions for `FEED_HORSE` and `IS_HAPPY_HORSE` to explicitly mention their input parameters, aligning them with the corresponding function declarations. This ensures clarity in the code and assists developers and auditors in understanding the expected input parameters for these functions.

Update code should look like this:

```huff
#define macro FEED_HORSE(horseId) = takes (1) returns (0) { 
    horseId                  // [horseId]
    timestamp               // [horseId, timestamp]
    STORE_ELEMENT(0x00)     // []

    // End execution 
    0x11 timestamp mod     
    endFeed jumpi                
    revert 
    endFeed:
    stop
}

#define macro IS_HAPPY_HORSE(horseId) = takes (1) returns (1) {
    horseId                          // [horseId]
    0x04 calldataload               // [horseId, horseFedTimestamp]
    LOAD_ELEMENT(0x00)               // [horseId, horseFedTimestamp]
    timestamp                        // [horseId, horseFedTimestamp, timestamp]
    dup2 dup2                        // [horseId, horseFedTimestamp, timestamp, horseFedTimestamp]
    sub                              // [horseId, timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
    [HORSE_HAPPY_IF_FED_WITHIN_CONST]// [horseId, HORSE_HAPPY_IF_FED_WITHIN, timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
    lt                               // [horseId, HORSE_HAPPY_IF_FED_WITHIN < timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
    start_return_true jumpi IS_HAPPY_HORSE  // [horseId, timestamp, horseFedTimestamp] if true jump to start_return_true else go to next line
    eq                               // [horseId, timestamp == horseFedTimestamp]
    start_return 
    jump
    
    start_return_true:
    0x01                             // [horseId, 1]

    start_return:
    // Store value in memory.
    0x00 mstore                      // [horseId]

    // Return value
    0x20 0x00 return                 // [horseId]
}
```
## <a id='L-04'></a>L-04. `IS_HAPPY_HORSE` wrong returns value in definition            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/8255173c9340c12501881c9ecdd4175ff7350f5d/src/HorseStore.huff#L93-L115

## Summary

The `IS_HAPPY_HORSE` macro has a discrepancy in its specification, as it is designed to return a boolean value to the stack, but the `returns` declaration in the macro definition inaccurately states `returns (0)` instead of `returns (1)`. This inconsistency could lead to confusion and potential errors in understanding the macro's behavior.

## Vulnerability Details

The `IS_HAPPY_HORSE` macro is intended to evaluate whether a horse has been fed within a certain time frame and return a boolean value accordingly. However, the `returns (0)` specification implies that the macro does not return any item to the stack, which is not aligned with its actual behavior.

## Impact

The incorrect specification in the `returns` declaration may cause confusion for developers and analysts trying to understand the macro's intended functionality. This discrepancy could lead to misinterpretations and potential implementation errors.

## Tools Used

Manual review.

## Recommendations

It is recommended to update the `returns` specification in the `IS_HAPPY_HORSE` macro definition to accurately reflect its behavior. The corrected version should be:

```huff
#define macro IS_HAPPY_HORSE() = takes (0) returns (1) {
    // ... (macro body)
}
```




