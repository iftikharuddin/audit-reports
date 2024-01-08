# Superform Protocol Report

Selected findings from Superform audit. [21 selected/confirmed till now, waiting for final results]

## [H-1] **Potential loss of ether in `registerAERC20`**

**Severity:** High

**Relevant GitHub Links:**
- [ERC1155A.sol - registerAERC20](https://github.com/superform-xyz/ERC1155A/blob/e7d53f306989ba205c779973d1b5e86755a1b9c0/src/ERC1155A.sol#L259-L267)

**Summary:**
The smart contract ERC1155A contains a potential vulnerability related to Ether handling. Specifically, the contract has payable functions but lacks a withdrawal mechanism. This can result in a loss of Ether sent to the contract.

**Vulnerability Details:**
The vulnerable function is `registerAERC20(uint256)` in ERC1155A.sol, which is marked as payable. However, the contract does not provide a function to withdraw Ether, posing a risk of fund lock-up.

**Impact:**
Without a withdrawal mechanism, users may lose access to Ether sent to the contract. This issue could lead to a degradation of user trust and financial loss.

**Recommendation:**
Option 1: Review the `registerAERC20` function to determine if it requires the payable attribute. If not, remove the payable attribute to prevent unintended Ether transfers.
Option 2: If the contract is intended to receive Ether, implement a secure withdrawal mechanism to allow users to retrieve their funds.

This modification enhances the security and usability of the contract, preventing potential fund lock-up and maintaining user trust.

## [L-1] **Floating Pragma Vulnerability in Superform Smart Contract**

**Severity:** LOW

**Relevant GitHub Links:**
- [ERC1155A.sol](https://github.com/superform-xyz/ERC1155A/blob/e7d53f306989ba205c779973d1b5e86755a1b9c0/src/ERC1155A.sol#L2)
- [aERC20.sol](https://github.com/superform-xyz/ERC1155A/blob/e7d53f306989ba205c779973d1b5e86755a1b9c0/src/aERC20.sol#L2)

**Summary:**
The Superform smart contract repository utilizes a floating pragma in several files, exposing the project to potential security risks. This deviation from the best practice of using a locked pragma version could lead to undiscovered vulnerabilities, code inconsistency, and overall security issues.

**Vulnerability Details:**
The floating pragma vulnerability is observed in multiple files within the Superform smart contract codebase. Instead of specifying a locked pragma version, a range of compiler versions is allowed, creating uncertainty about the exact version used for deployment. This practice deviates from recommended security standards and could pose a significant risk to the project.

**Impact:**
The impact of the floating pragma vulnerability includes the following potential risks:
- Using a very recent compiler version may expose the code to undiscovered vulnerabilities.

**Tools Used:**
Manual review

**Recommendations:**
1. **Use a Strict and Locked Pragma Version:**
   Update all Solidity files in the Superform smart contract codebase to use a strict and locked pragma version. This helps ensure consistency and avoids potential vulnerabilities associated with floating pragma.

2. **Prefer a Stable Compiler Version:**
   Choose a compiler version that is neither too old nor too recent, striking a balance between stability and security. This minimizes the risk of using a version with unresolved bugs or undiscovered vulnerabilities.

By implementing these recommendations, the Superform project can enhance the security of its smart contract codebase, mitigate potential vulnerabilities, and foster a more robust and reliable blockchain application.

Certainly, here is the information formatted for your GitHub portfolio:

Certainly! Here's a formatted version for your GitHub:

## [L-2] **Assuming Oracle Price Precision can lead to Inaccurate Gas Calculation Fees in `_convertToNativeFee`**

**Severity:** Low

**Relevant GitHub Links:** [PaymentHelper.sol - Lines 798-819](https://github.com/superform-xyz/superform-core/blob/29aa0519f4e65aa2f8477b76fc9cc924a6bdec8b/src/payments/PaymentHelper.sol#L798-L819)

**Summary:**
In the `PaymentHelper` contract, the function `_convertToNativeFee` is utilized for xChain gas estimation. However, it currently relies on non-ETH pairs, assuming a decimal precision of 8. Considering Superform's potential deployment on various EVM chains, issues may arise with compatibility if deployed on chains where price feeds have 18 decimals or if using ETH pairs requiring precision up to 18 decimals.

**Vulnerability Details:**
The issue stems from assuming a fixed decimal precision (8 decimals) for price feeds, which may not hold true for all chains and pairs. Different chains or specific pairs might have varying decimal precisions. For instance, non-ETH pairs typically use 8 decimals, while ETH pairs use 18 decimals.

**Impact:**
This issue could lead to inaccurate gas estimation if Superform is deployed on chains with different decimal precision for price feeds or if it utilizes ETH pairs that require 18 decimals.

**Tools Used:**
Manual review

**Recommendations:**
Smart contracts should dynamically determine the decimal precision for the relevant price feed by calling `AggregatorV3Interface.decimals()` to ensure compatibility with different chains and pairs. This approach will prevent assumptions about a fixed decimal precision and accommodate various scenarios, enhancing the robustness and flexibility of the system.

## [I-1] **Gas Efficiency Improvement in ERC1155A Contract**

**Severity:** Informational

**Relevant GitHub Links:**
- [ERC1155A.sol - setApprovalForOne](https://github.com/superform-xyz/ERC1155A/blob/e7d53f306989ba205c779973d1b5e86755a1b9c0/src/ERC1155A.sol#L180)
- [ERC1155A.sol - increaseAllowance](https://github.com/superform-xyz/ERC1155A/blob/e7d53f306989ba205c779973d1b5e86755a1b9c0/src/ERC1155A.sol#L191)
- [ERC1155A.sol - decreaseAllowance](https://github.com/superform-xyz/ERC1155A/blob/e7d53f306989ba205c779973d1b5e86755a1b9c0/src/ERC1155A.sol#L198)

**Summary:**
The functions `setApprovalForOne`, `increaseAllowance`, and `decreaseAllowance` in the ERC1155A contract declare `address owner = msg.sender` within the function body, introducing potential gas overhead. These local variables are used only once and can be omitted for improved gas efficiency.

**Vulnerability Details:**
In the mentioned functions, `address owner = msg.sender` is declared within the function body. These variables are used only once and are not essential for the functions' logic. The use of local variables in this context may result in slightly higher gas costs due to repeated variable declarations, particularly if the functions are called iteratively.

**Impact:**
The impact of this issue is low. The gas inefficiency introduced by the unnecessary declaration of `address owner = msg.sender` is minor. However, removing these variables can lead to slightly more gas-efficient code.

**Tools Used:**
Manual Review

**Recommendations:**
It is recommended to remove the declarations `address owner = msg.sender` in the `setApprovalForOne`, `increaseAllowance`, and `decreaseAllowance` functions and use `msg.sender` directly within the corresponding functions. This adjustment improves gas efficiency, especially in scenarios where these functions are called iteratively or within loops.

**Example:**
```solidity
function setApprovalForOne(address spender, uint256 id, uint256 amount) public virtual {
    _setApprovalForOne(msg.sender, spender, id, amount);
}

function increaseAllowance(address spender, uint256 id, uint256 addedValue) public virtual {
    _increaseAllowance(msg.sender, spender, id, addedValue);
}

function decreaseAllowance(address spender, uint256 id, uint256 subtractedValue) public virtual {
    _decreaseAllowance(msg.sender, spender, id, subtractedValue);
}
```

This modification optimizes gas costs and maintains code simplicity, making the functions more efficient, particularly in situations where they might be called repeatedly.

## [I-2] **Dynamic Interface ID Generation in supportsInterface Function**

**Severity:** Informational

**Relevant GitHub Links:**
- [ERC1155A.sol - supportsInterface](https://github.com/superform-xyz/ERC1155A/blob/e7d53f306989ba205c779973d1b5e86755a1b9c0/src/ERC1155A.sol#L368-L371)

**Summary:**
The `supportsInterface` function in the ERC1155A contract contains hardcoded values for ERC165 interface IDs. While this implementation is functional, it introduces a potential maintenance risk, as any changes to the ERC165 standard may not be reflected in this hardcoded list.

**Vulnerability Details:**
The function uses hardcoded values (`0x01ffc9a7`, `0xd9b67a26`, and `0x0e89341c`) to check for support of ERC165, ERC1155, and ERC1155MetadataURI interfaces, respectively. Hardcoding these values may result in future compatibility issues if the ERC165 standard or related standards are updated.

**Impact:**
The impact of this issue is low. The hardcoded values are functional for the current state of ERC165 and ERC1155 standards. However, there is a risk of potential future issues if the standards evolve, and these values are not updated accordingly.

**Tools Used:**
Manual Review

**Recommendations:**
It is recommended to dynamically generate the interface IDs using the `type` keyword or other appropriate methods instead of hardcoding them. This approach ensures that the function remains compatible with any future changes to the ERC165 standard or related standards.

**Example:**
```solidity
function supportsInterface(bytes4 interfaceId) public view virtual returns (bool) {
    return interfaceId == type(IERC165).interfaceId
        || interfaceId == type(IERC1155).interfaceId
        || interfaceId == type(IERC1155MetadataURI).interfaceId;
}
```

By using the `type` keyword, the interface IDs are generated at compile time, providing a more robust and future-proof solution. This approach aligns with best practices for ensuring contract compatibility with evolving standards.

## [I-3] **Consistency Enhancement in LiFiValidator Contract**

**Severity:** Informational

**Relevant GitHub Links:**
- [LiFiValidator.sol - decodeAmountIn](https://github.com/superform-xyz/superform-core/blob/29aa0519f4e65aa2f8477b76fc9cc924a6bdec8b/src/crosschain-liquidity/lifi/LiFiValidator.sol#L121-L149)

**Summary:**
In the `decodeAmountIn` function of the LiFiValidator contract, there is a discrepancy between the comment and the actual variable name returned from the function. The comment mentions "amountIn," while the variable returned is named "amount_."

**Vulnerability Details:**
The inconsistency in variable naming and comments can lead to confusion and misunderstandings for developers or reviewers. It doesn't pose a direct security threat but may result in code readability issues.

**Impact:**
Reduced code readability and potential confusion for developers.

**Tools Used:**
Manual Review

**Recommendations:**
Update the comment in the `decodeAmountIn` function to mention "amount_" instead of "amountIn" to maintain consistency between comments and code. Alternatively, consider renaming the variable to "amountIn" if it better reflects the intended meaning.

This change will enhance code clarity and make it more understandable for anyone reviewing or maintaining the code.


## [I-4] **Enhancement for Code Readability in DstSwapper Contract**

**Severity:** Informational

**Relevant GitHub Links:**
- [DstSwapper.sol - Numeric Literals](https://github.com/superform-xyz/superform-core/blob/29aa0519f4e65aa2f8477b76fc9cc924a6bdec8b/src/crosschain-liquidity/DstSwapper.sol#L353)

**Summary:**
The code currently utilizes numeric literals (magic numbers) such as `10_000` without named constants.

**Vulnerability Details:**
The code contains numeric literals without using named constants. For example, the value `10_000` is used without clear context. While this does not pose a direct security risk, it can impact code readability and maintainability over time.

**Impact:**
The absence of named constants for numeric literals may result in decreased code readability and make it challenging for developers to understand the significance of these values. It does not directly impact the security of the contract.

**Tools Used:**
Manual Review

**Recommendations:**
1. **Named Constants:**
   Consider replacing numeric literals, such as `10_000`, with named constants. This practice enhances code readability and provides meaningful context for the purpose of specific values. For instance, defining `SLIPPAGE_THRESHOLD` with the value `10_000` would improve the clarity of the code.

   **Example:**
   ```solidity
   // SPDX-License-Identifier: BUSL-1.1
   pragma solidity ^0.8.23;

   contract DstSwapper {
       uint256 private constant SLIPPAGE_THRESHOLD = 10_000;
       // Other contract code...
   }

   if (v.balanceDiff < ((v.expAmount * (SLIPPAGE_THRESHOLD - v.maxSlippage)) / SLIPPAGE_THRESHOLD)) {
       revert Error.SLIPPAGE_OUT_OF_BOUNDS();
   }
   ```

This modification enhances code readability, making it easier for developers to understand the purpose of specific numeric values.

Certainly, here is the information formatted for your GitHub portfolio:

## [I-5] **Documentation Enhancement in SocketValidator Contract**

**Severity:** Informational

**Relevant GitHub Links:**
- [SocketValidator.sol - decodeSwapOutputToken](https://github.com/superform-xyz/superform-core/blob/29aa0519f4e65aa2f8477b76fc9cc924a6bdec8b/src/crosschain-liquidity/socket/SocketValidator.sol#L97-L99)

**Summary:**
The SocketValidator contract exhibits a minor issue related to the documentation of the `decodeSwapOutputToken` function. The function includes a revert statement with a specific error message, which may be intentional for interface enforcement. However, adding a comment to explain the purpose of the revert statement would enhance clarity for developers.

**Vulnerability Details:**
The `decodeSwapOutputToken` function contains a revert statement with a specific error message (`Error.CANNOT_DECODE_FINAL_SWAP_OUTPUT_TOKEN()`). While this could be an intentional part of the interface, adding a comment would help developers understand the purpose of the revert statement and guide them on the expected behavior.

**Impact:**
This informational issue has no direct impact on the security of the contract. It is a suggestion for improved documentation to enhance code readability and provide guidance to developers implementing the interface.

**Tools Used:**
Manual Review

**Recommendations:**
It is recommended to add a comment to the `decodeSwapOutputToken` function to explain the purpose of the revert statement and guide developers on the expected behavior. This will contribute to a more transparent and developer-friendly codebase.

**Example:**
```solidity
/// @notice To be implemented by contracts that handle the decoding of the final swap output token.
/// @dev Implementers should replace the revert statement with the appropriate logic for decoding the output token.
function decodeSwapOutputToken(bytes calldata /*txData_*/) external pure returns (address /*token_*/);
```

This modification improves the documentation, providing clear guidance for developers implementing the interface.

## [I-6] **Code Organization Enhancement in CoreStateRegistry Contract**

**Severity:** Informational

**Relevant GitHub Links:**
- [CoreStateRegistry.sol - validateSlippage](https://github.com/superform-xyz/superform-core/blob/29aa0519f4e65aa2f8477b76fc9cc924a6bdec8b/src/crosschain-data/extensions/CoreStateRegistry.sol#L490-L497)

**Summary:**
The `validateSlippage` function is categorized as a public view function but is currently placed within the internal functions section. This discrepancy may lead to confusion and impacts the code's readability and maintainability.

**Vulnerability Details:**
The issue arises from the misplacement of the `validateSlippage` function in the internal functions section instead of the public functions section. As a result, external developers reviewing the code may find it challenging to locate and understand the purpose of this publicly accessible function.

**Impact:**
The impact of this issue is primarily related to code organization, readability, and developer experience. It does not introduce a security vulnerability but could lead to confusion and hinder effective code comprehension.

**Tools Used:**
Manual Review

**Recommendations:**
Move the `validateSlippage` function to the public functions section to align its visibility with its intended usage as a publicly accessible view function. This adjustment improves code organization and contributes to a more straightforward and understandable codebase.