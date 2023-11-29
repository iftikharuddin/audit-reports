# Audit of Converagence Protocol

The Convergence audit in Sherlock:

### Issue #1: Unsafe use of transfer() in burnPosition

## **Summary:**
The `burnPosition` function in `LockingPositionService` contains a potential vulnerability related to the direct use of the `transfer` function for the CVG token. This operation might not revert on failure but instead returns a boolean value. The code lacks a check on the success of the transfer, which could potentially lead to funds being lost or the operation being considered successful even if it fails.

## **Vulnerability Detail:**
In the `burnPosition` function, the direct use of the transfer function might not revert on failure but instead returns a boolean value. The code lacks a check on the success of the transfer, which could potentially lead to funds being lost or the operation being considered successful even if it fails.

## **Impact:**
If the CVG token's transfer function fails, the user may lose funds, and the failure might go unnoticed, affecting the overall functionality and security of the contract.

## **Code Snippet:**
[Link to Code](https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L511)

```solidity
cvg.transfer(msg.sender, totalCvgLocked); // @audit this line has an issue
```

## **Tool used:**
Manual Review

## **Recommendation:**
It is recommended to update the token transfer operation in the `burnPosition` function to use a pattern that checks for success and reverts if the transfer fails. Consider adopting a pattern similar to OpenZeppelin's SafeERC20 library:

```solidity
bool success = cvg.transfer(msg.sender, totalCvgLocked);
require(success, "CVG transfer failed");
```

---

### Issue #2: Potential Security Vulnerability in onlyWalletOrWhiteListedContract Modifier of LockingPositionService Contract due to tx.origin Usage

## **Summary:**
The use of `tx.origin` in `LockingPositionService` allows any caller, including potential malicious actors. To enhance security, it is recommended to replace `tx.origin` with `msg.sender`, as the latter provides the direct caller's address. While `tx.origin` may be semi-legitimized for tracking contract interactions, its use is discouraged due to security risks.

## **Vulnerability Detail:**
Insecure usage of `tx.origin` in `LockingPositionService` contract's `onlyWalletOrWhiteListedContract` modifier may expose security vulnerabilities, allowing potential manipulation by malicious actors.

## **Impact:**
The vulnerability in the `onlyWalletOrWhiteListedContract` modifier of the `LockingPositionService` contract using `tx.origin` can lead to unauthorized access, compromising the security of the contract.

## **Code Snippet:**
[Link to Code](https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L184)

```solidity
require(msg.sender == tx.origin || isContractLocker[msg.sender], "NOT_CONTRACT_OR_WL");
```

## **Tool used:**
Manual Review

## **Recommendation:**
Remove `msg.sender == tx.origin` from the require check in `_onlyWalletOrWhiteListedContract`. The updated code ensures that the function and modifier solely rely on `msg.sender` for enhanced security:

```solidity
require(isContractLocker[msg.sender], "NOT_CONTRACT_OR_WL");
```