# First Flight #8: Math Master - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Wrong value in checks of the `lt` of a right shift](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Lack of free memory pointer management, potential overrite & out of gas errors](#M-01)
- ## Low Risk Findings
    - ### [L-01. Unnecessary rounding adjustment in `mulWadUp` function](#L-01)
    - ### [L-02. Custom errors will not work in 0.8.3 version of solidity](#L-02)
    - ### [L-03. Incorrect error selector in `mulWadUp` function of MathMasters Library](#L-03)
    - ### [L-04. Function selector is wrong for `MathMasters__MulWadFailed` in `mulWad`](#L-04)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #8

### Dates: Jan 25th, 2024 - Feb 2nd, 2024

[See more contest details here](https://www.codehawks.com/contests/clrp8xvh70001dq1os4gaqbv5)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 1
   - Low: 4


# High Risk Findings

## <a id='H-01'></a>H-01. Wrong value in checks of the `lt` of a right shift            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L77

## Summary
The code contains a discrepancy in the `sqrt` function's comment, where the correct value of `16777215` is mentioned, but the code incorrectly uses `16777002`. This inconsistency may lead to incorrect comparisons in the algorithm, affecting the initial estimate provided by the Babylonian method.

## Vulnerability Details
The specific issue arises is present in this code:
```solidity
r := or(r, shl(4, lt(16777002, shr(r, x))))
```
where `16777002` should be corrected to `16777215` to accurately reflect the maximum value resulting from a right shift.

## Impact
The impact of this issue lies in the potential misalignment of the initial estimate provided by the Babylonian method. Incorrect comparisons may lead to unexpected behavior, affecting the convergence speed of the algorithm.

The significance of this issue lies in the fact that the value `16777002` does not accurately represent the maximum value that can result from a right shift. Specifically, when `r` is right-shifted by `x` bits, the maximum value it can have is `0xffffff`, not `0xffff2a`. Therefore, the `lt` (less than) comparison may produce incorrect results and potentially lead to unexpected behavior.

## Tools Used
Manual code review and analysis.

## Recommendations
Use correct decimal value in the code:

```diff
-lt(16777002, shr(r, x))
+lt(16777215, shr(r, x))
```

		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Lack of free memory pointer management, potential overrite & out of gas errors            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L53

https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L40


## Summary
The `MathMasters`  library uses the `mstore` instruction to manipulate values in memory, including writing custom error codes to the solidity free memory pointer's position (0x40). However, it does not directly interact with the value at memory position 0x40, which represents the "free memory pointer" in Solidity.

## Vulnerability Details
The library writes custom error codes to the free memory pointer's position without explicitly managing the free memory pointer itself. It does not include instructions to load or update this pointer in its functions. Proper memory management practices, including loading and updating the free memory pointer, are crucial when using inline assembly operations. Neglecting to follow these practices may lead to unintended consequences, such as overwriting data or corrupting the memory layout.

## Impact
Failure to manage the free memory pointer appropriately may result in unexpected behavior, data corruption, or security vulnerabilities. If a contract using this library neglects to load and update the free memory pointer correctly, it may lead to memory allocation issues and compromise the integrity of the contract's data. Along with this it might have potential out of gas errors if the call data doesn't fit in the scratch area which you are overwriting the free memory pointer

## Tools Used

Manual review.

## Recommendations
**Free Memory Pointer Management:** The library should use the proper sequence of loading and updating the free memory pointer before and after executing inline assembly operations. This ensures correct memory allocation and prevents unintended overwrites.



# Low Risk Findings

## <a id='L-01'></a>L-01. Unnecessary rounding adjustment in `mulWadUp` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L56

## Summary
There is a rounding adjustment in the `mulWadUp` function, which appears to be unnecessary given the existing rounding logic. This additional rounding adjustment has been identified as the cause of an assertion failure during fuzz testing. 

## Vulnerability Details
In the `mulWadUp` function, the following code snippet is unnecessary:

```solidity
if iszero(sub(div(add(z, x), y), 1)) {
    x := add(x, 1)
}
```

This code attempts a rounding adjustment, but the existing rounding logic in the function effectively handles rounding up after the multiplication. The unnecessary adjustment is causing assertion failures during fuzz testing.

## Impact
This unnecessary rounding adjustment is leading to assertion failures. However, in normal execution scenarios, it does not contribute to the correct functionality of the rounding process. Removing this unnecessary code will likely result in cleaner and more efficient code execution.

## POC

- Make sure to configure foundry.toml `runs = 1000000`
- Run the fuzz test on original code and you will get this result

```
Failing tests:
Encountered 1 failing test in test/MathMasters.t.sol:MathMastersTest
[FAIL. Reason: assertion failed; counterexample: calldata=0xfb6ea94f00000000000000000000000000000000000000000002bfc66f73219d55eb3274000000000000000000000000000000000000000000015fe337b990ceaaf5993b args=[3323484123583475243233908 [3.323e24], 1661742061791737621616955 [1.661e24]]] testMulWadUpFuzz(uint256,uint256) (runs: 3979, μ: 895, ~: 1146)
```

- Now remove/comment the unnecessary rounding code and run the fuzzer via cmd ` orge test --match-test testMulWadUpFuzz -vvvvv`

Result:
```
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 65.07s
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

- Foundry fuzzing

## Recommendations

The following recommendations are provided:

**Remove Unnecessary Rounding Adjustment:**
   The unnecessary rounding adjustment code should be removed from the `mulWadUp` function to streamline the logic and prevent assertion failures during fuzz testing.

```diff
/// @dev Equivalent to `(x * y) / WAD` rounded up.
function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
    /// @solidity memory-safe-assembly
    assembly {
        // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
        //Overflow Check
        if mul(y, gt(x, or(div(not(0), y), x))) {
            // extra x in end? and what is or used for?
            mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`. 
            revert(0x1c, 0x04)
        }
        // Rounding Adjustment
-        if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) } 
        z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
    }
}
## <a id='L-02'></a>L-02. Custom errors will not work in 0.8.3 version of solidity            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L3


## Summary
Custom errors were introduced in Solidity version 0.8.4. However, the specified Solidity version in the pragma statement is 0.8.3, leading to a compilation error. 

## Vulnerability Details
The MathMasters defines custom errors using the `error` keyword, a feature introduced in Solidity version 0.8.4. However, the specified pragma statement at the beginning of the contract (`pragma solidity ^0.8.3;`) indicates an older Solidity version that does not support custom errors. As a result, attempting to compile the contract with the specified version will lead to a compilation error.

## Impact
The contract uses features introduced in Solidity version 0.8.4, while being constrained by the version specified in the pragma statement. This discrepancy may hinder the intended functionality of the contract and prevent successful deployment on a version 0.8.3 compiler.

## Recommendations
Consider upgrading the Solidity version specified in the pragma statement to `0.8.4` or a later version that supports custom errors. This can be achieved by updating the pragma statement at the beginning of the contract:

   ```solidity
   pragma solidity ^0.8.4;
   ```


## <a id='L-03'></a>L-03. Incorrect error selector in `mulWadUp` function of MathMasters Library            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L53

## Summary
The `mulWadUp` function in the MathMasters library incorrectly references the `MathMasters__MulWadFailed` error with the selector `0xbac65e5b` instead of the correct selector `0xa56044f7`. This misalignment may lead to confusion and unexpected behavior.

## Vulnerability Details
In the assembly code of the `mulWadUp` function, the error is specified with the incorrect selector, potentially causing issues in error handling and reporting. 

## Impact
This discrepancy in the error selector may result in misidentifying errors related to the `mulWadUp` function, affecting the accuracy of error handling. Developers relying on the correct error selector may face challenges in debugging and understanding the root cause of potential failures.

## POC
- Run this cmd `forge inspect MathMasters errors` and you will see the selector for `MathMasters__MulWadFailed` is `a56044f7` not `0xbac65e5b`

```solidity
{
  "MathMasters__MulWadFailed()": "a56044f7"
}
```
## Recommendations
Update the selector for the `MathMasters__MulWadFailed` error in the `mulWadUp` function to the correct value (`0xa56044f7`). Ensure consistency between error definitions and their associated selectors for accurate error identification and reporting.

Update code should be:

```solidity
function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
    /// @solidity memory-safe-assembly
    assembly {
        // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
        if mul(y, gt(x, or(div(not(0), y), x))) {
            mstore(0x40, a56044f7) // `MathMasters__MulWadFailed()`
            revert(0x1c, 0x04)
        }
        if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
        z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
    }
}
```
## <a id='L-04'></a>L-04. Function selector is wrong for `MathMasters__MulWadFailed` in `mulWad`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/84c149baf09c1558d7ba3493c7c4e68d83e7b3aa/src/MathMasters.sol#L40

## Summary

The issue involves the usage of an incorrect function selector for the `MathMasters__MulWadFailed()` custom error. The current selector is `0xbac65e5b`, but the correct one is `0xa56044f7`. This inconsistency may lead to confusion and potential errors in error handling.

## Vulnerability Details

The `mulWad` function in the `MathMasters` library has an incorrect function selector for the `MathMasters__MulWadFailed()` custom error. The incorrect selector `0xbac65e5b` is used, while the correct selector is `0xa56044f7`.

```solidity
mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`. // @audit wrong selector
revert(0x1c, 0x04)
```

## Impact

The incorrect function selector could result in misinterpretation and misidentification of the error during execution. Developers relying on the accurate identification of errors may encounter challenges in handling this specific error case.

## POC

- When I run `forge inspect MathMasters errors` I get the below result for `MathMasters__MulWadFailed` but the currently used one is wrong.

```
{
  "MathMasters__MulWadFailed()": "a56044f7"
}
```

## Recommendations

It is recommended to update the function selector in the `mulWad` function to the correct one (`0xa56044f7`). This ensures consistency and accuracy in error handling. Additionally, a thorough review of other function selectors in the library is advised to verify their correctness.

```solidity
mstore(0x40, 0xa56044f7) // `MathMasters__MulWadFailed()`.
revert(0x1c, 0x04)
```


