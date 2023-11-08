# First Flight #1: PasswordStore - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Password Handling in Deployment Script](#H-01)
    - ### [H-02. Storage of Plain-Text Passwords](#H-02)
    - ### [H-03. Access Control Lacking in setPassword Function](#H-03)

- ## Low Risk Findings
    - ### [L-01. Revert Usage in getPassword Function is NOT gas efficient](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #1

### Dates: Oct 18th, 2023 - Oct 25th, 2023

[See more contest details here](https://www.codehawks.com/contests/clnuo221v0001l50aomgo4nyn)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 0
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Password Handling in Deployment Script            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/script/DeployPasswordStore.s.sol#L11

## Summary

In the DeployPasswordStore.s.sol script, it is not recommended to pass plain password text/values directly.

## Vulnerability Details

Passing plain password text/values directly in the `DeployPasswordStore.s.sol` script is insecure and not recommended, potentially exposing sensitive information and compromising overall security.

## Impact

Passing plain password text/values directly in the `DeployPasswordStore.s.sol` script can expose sensitive information, posing a security risk to the overall application and potentially leading to unauthorized access and data breaches.

## Tools Used

- Foundry
- Manual testing

## Recommendations

A more secure approach would be to read passwords from an environment file (e.g., .env).

- create `.env` file and configure `PASSWORD` in there
- access it `DeployPasswordStore.s.sol` like `process.env.PASSWORD`

```diff
-passwordStore.setPassword("myPassword");
+passwordStore.setPassword(process.env.PASSWORD);
```

## <a id='H-02'></a>H-02. Storage of Plain-Text Passwords            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L27C12-L27C12

## Summary

In the `PasswordStore` contract, the `setPassword` function stores plain-text passwords, which is a severe security risk.

## Vulnerability Details

The PasswordStore contract's `setPassword` function stores passwords in plain text, posing a severe security risk due to potential unauthorized access and data breaches.

## Impact

Storing plain-text passwords in the setPassword function of the PasswordStore contract poses a significant security risk.

## Tools Used

- Foundry
- Manual testing

## Recommendations

Secure hashing and salting should be implemented to protect user data.

```diff
-function setPassword(string memory newPassword) external {
-  s_password = newPassword;
-  emit SetNetPassword();
-}
+function setPassword(string memory newPassword) onlyOwner external {
+  s_passwordHash = keccak256(abi.encodePacked(newPassword, salt));
+  emit SetNetPassword();
+}

Make sure to define `bytes32 private salt;` and assign `salt = Random.generateRandomBytes32();` in constructor and for Random use `import "@openzeppelin/contracts/utils/cryptography/Random.sol";`




## <a id='H-03'></a>H-03. Access Control Lacking in setPassword Function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L26C5-L30C1

## Summary

The `setPassword` function in the `PasswordStore` contract lacks proper access control modifiers or owner checks, making it vulnerable to unauthorized access.

## Vulnerability Details

The vulnerability resides in the setPassword function of the PasswordStore contract, where the absence of access control modifiers or owner checks allows unrestricted access. This jeopardizes the contract's security, potentially leading to unauthorized alterations of sensitive data. Implementing access control or onlyOwner modifiers is essential to safeguard the contract.

## Impact

Absence of an `onlyOwner` modifier or owner checks in the `setPassword` function of the `PasswordStore` contract poses a significant risk, enabling unauthorized access and potential data breaches, emphasizing the need for robust access control implementation.

## Tools Used

The audit was conducted using manual code review and best practices in smart contract security. 

## Recommendations

Implement access control or owner checks in the `setPassword` function of the `PasswordStore` contract to enhance security and restrict access to authorized parties. This essential measure safeguards sensitive data, strengthening the contract's security and mitigating potential vulnerabilities.

```diff
- function setPassword(string memory newPassword) external {
+ function setPassword(string memory newPassword) external onlyOwner {
```       






		


# Low Risk Findings

## <a id='L-01'></a>L-01. Revert Usage in getPassword Function is NOT gas efficient            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L36

## Summary

Optimize the `getPassword` function in the `PasswordStore` contract by replacing the revert statement with a gas-efficient access control modifier, improving code efficiency and readability.

## Vulnerability Details

- In the `PasswordStore` contract, replace the use of `revert` in the getPassword function with a gas-efficient modifier for access control.

## Impact

Enhancing the `getPassword` function of the `PasswordStore` contract with a gas-efficient access control modifier improves efficiency, readability, security, and user-friendliness, while reducing gas consumption.

## Tools Used

- Foundry
- Manual testing

## Recommendations

Enhance code readability and efficiency by implementing an access control modifier for owner-restricted functions.

```diff
-function getPassword() external view returns (string memory) {
- if (msg.sender != s_owner) {
-    revert PasswordStore__NotOwner();
- }
- return s_password;
-}

+function getPassword() onlyOwner external view returns (string memory) {
+   return s_password;
+}

```


