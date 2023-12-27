# Audting Superform Protocol

Selected findings from Superform audit.

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