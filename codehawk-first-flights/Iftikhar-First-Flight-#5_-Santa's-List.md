# First Flight #5: Santa's List - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Non SantaTokens holders can call `buyPresent`, potentially resulting in negative balances & an unintended reduction in the total supply of SantaTokens](#H-01)
    - ### [H-02. FFI is enabled by default, anyone can run arbitrary commands](#H-02)
    - ### [H-03. Timestamp Reliance Enables Premature Reward Collection in `SantasList`](#H-03)
    - ### [H-04. Incorrect Christmas 2023 Block Time Allows Early Calling of `collectPresent` Function](#H-04)
    - ### [H-05. Mismatched Cost in buyPresent Function, Allows for Lower-Cost Exploitation](#H-05)
    - ### [H-06. Critical Security Vulnerability in transferFrom Function Allows Unauthorized Token Transfers](#H-06)
    - ### [H-07. Unrestricted Access to checkList Function Allows Unauthorized Status Changes](#H-07)
- ## Medium Risk Findings
    - ### [M-01. Vulnerability in `collectPresent` Allows Unauthorized Status Manipulation](#M-01)
- ## Low Risk Findings
    - ### [L-01. Missing 'indexed' Fields in Events](#L-01)
    - ### [L-02. Insecure Timestamp Dependency and Arbitrum Compatibility in Smart Contract](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #5

### Dates: Nov 30th, 2023 - Dec 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clpba0ama0001ywpabex01hrp)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 7
   - Medium: 1
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Non SantaTokens holders can call `buyPresent`, potentially resulting in negative balances & an unintended reduction in the total supply of SantaTokens            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L172-L175

## **Summary**

A critical vulnerability was identified in the `SantsList::buyPresent` function. The function allows burning tokens without properly checking whether the caller holds SantaTokens. This omission can lead to unintended consequences, including negative balances and a reduction in the total supply of SantaTokens.

## **Vulnerability Details**

### **Function: burn(address from)**

The `buyPresent` function allows unauthorized callers to burn tokens without verifying SantaTokens ownership, potentially leading to negative balances and a decrease in total supply.

## **Impact**

The absence of a check for SantaTokens holders in the `buyPresent` function allows any address to call the function, potentially resulting in negative balances and an unintended reduction in the total supply of SantaTokens.

## **Tools used**

- Manual review

## **Recommendations**

**Add SantaTokens Holder Check:**
   Implement a check in the `burn` function to ensure that the caller (`msg.sender`) holds SantaTokens before allowing the burning process. This can be achieved by verifying the balance of SantaTokens for the specified address.

   ```solidity
   require(i_santaToken.balanceOf(msg.sender) > 0, "Caller does not have any SantaTokens");
   ```

## <a id='H-02'></a>H-02. FFI is enabled by default, anyone can run arbitrary commands            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/foundry.toml#L9

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/test/mocks/CheatCodes.t.sol

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/test/unit/SantasListTest.t.sol#L149-L154

### **Summary**
The SantasListTest `testPwned` function employs the Foreign Function Interface (FFI) to execute commands. The project configuration (`foundry.toml`) reveals that FFI is enabled. The interface `_CheatCodes` declares a function allowing arbitrary commands execution. This raises significant security concerns regarding the potential manipulation of tests and execution of malicious commands.

### **Vulnerability Details**
1. **FFI Usage in `testPwned` Function:**
   - The `testPwned` function utilizes FFI, providing a gateway for executing arbitrary commands during tests.
   - Lack of proper input validation and access controls in the FFI execution may lead to security vulnerabilities.

2. **Configuration (`foundry.toml`):**
   - The `foundry.toml` configuration file indicates FFI is enabled, increasing the risk of arbitrary command execution.

3. **Interface `_CheatCodes`:**
   - The `_CheatCodes` interface declares the function `ffi(string[] calldata) external returns (bytes memory);`.
   - The function signature allows for arbitrary command execution if FFI is enabled.

### **Impact**
The security implications include potential:
   - Manipulation of test results.
   - Execution of arbitrary commands, leading to data loss or service disruption.
   - Unauthorized access to sensitive information in testing environments.

### **Tools Used**
- Manual Review

### **Recommendations**
1. **Disable FFI by Default:**
   - Change the default configuration in `foundry.toml` to disable FFI, reducing the risk of arbitrary command execution.

2. **Access Controls:**
   - Implement robust access controls to restrict who can enable FFI and execute arbitrary commands.

3. **Secure Defaults:**
   - Enable FFI only when absolutely necessary and as a last resort. Avoid enabling it by default to prevent misuse.

4. **Monitoring:**
   - Implement thorough monitoring mechanisms to track FFI usage and command execution.

5. **Educational Guidance:**
   - Provide clear documentation and educational guidance on the risks associated with enabling FFI and executing arbitrary commands.


## <a id='H-03'></a>H-03. Timestamp Reliance Enables Premature Reward Collection in `SantasList`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L147-L166

## **Summary**
The `collectPresent` function in SantasList.sol is susceptible to frontrunning attacks, allowing miners to exploit time-dependent conditions for potential financial gain and other advantages.

## **Vulnerability Details**
The vulnerability arises from the use of `block.timestamp` to check if it's Christmas before allowing users to collect presents. Miners can manipulate the timestamp to make it appear as if Christmas has arrived, enabling them to front-run transactions and collect rewards prematurely.

## **Impact**
- Miners can exploit the vulnerability to front-run transactions and trigger reward mechanisms before the intended time.
- Users may experience unfair treatment, and the integrity of time-dependent conditions in the smart contract is compromised.

## **Tools used**
- Manual review

## **Recommendations**
   - Refrain from relying solely on `block.timestamp` for time-sensitive conditions. Consider using block numbers or a combination of block numbers and timestamps.
   - Consider using commit-reveal schemes, especially in scenarios where confidentiality is crucial. This adds an additional layer of security by introducing a delay between commitment and revelation

   - Incorporate randomness into the smart contract logic to make it more challenging for miners to predict outcomes and manipulate transactions.

   - Clearly communicate the potential risks associated with frontrunning and timestamp manipulation to users. Provide guidelines for secure interaction with the smart contract.

Implementing these recommendations will enhance the resilience of the smart contract against frontrunning attacks and improve overall security.

## <a id='H-04'></a>H-04. Incorrect Christmas 2023 Block Time Allows Early Calling of `collectPresent` Function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L147-L150

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L86

## **Summary**
The `collectPresent` function can be called before Christmas 2023 due to a discrepancy between the block timestamp condition and the actual Christmas 2023 block time.

## **Vulnerability Details**
The contract has a condition in the `collectPresent` function to prevent calls before Christmas 2023:
```solidity
if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
    revert SantasList__NotChristmasYet();
}
```
However, the `CHRISTMAS_2023_BLOCK_TIME` is set to `Thursday, December 22, 2022, 10:33:01 UTC`, which is not the expected Christmas date. This allows users to call the `collectPresent` function even before the actual Christmas date.

## **Impact**
Users can exploit the contract by calling the `collectPresent` function before the intended Christmas date, leading to undesired behavior and potentially affecting the distribution of presents.

## **Tools used**
- Manual review

## **Recommendations**
The `CHRISTMAS_2023_BLOCK_TIME` should be set to the actual expected Christmas date in 2023, ensuring that the `collectPresent` function can only be called within the specified time frame.
## <a id='H-05'></a>H-05. Mismatched Cost in buyPresent Function, Allows for Lower-Cost Exploitation            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L172-L175

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantaToken.sol#L28-L33

## Summary
The contract contains a discrepancy in the cost calculation for purchasing presents. The intended cost is specified as 2e18, but the `burn` function, which is responsible for deducting the cost from the user, is currently burning 1e18. This mismatch may allow users to exploit the contract and purchase presents at a lower cost than intended.

## Vulnerability Details
The vulnerability lies in the `burn` function, which is used to deduct SantaTokens when a user purchases a present. The parameter passed to `_burn` is inconsistent with the intended cost, leading to a mismatch between the intended cost and the actual deduction.

## Impact
If exploited, users could purchase presents at a lower cost than intended, potentially disrupting the economic balance of the contract.

## Tools Used
- Manual Review

## Recommendations
- Update the `burn` function's parameter in `SantaToken` to match the intended cost (2e18).

```solidity
function burn(address from) external {
    if (msg.sender != i_santasList) {
        revert SantaToken__NotSantasList();
    }
    _burn(from, 2e18);
}
```
## <a id='H-06'></a>H-06. Critical Security Vulnerability in transferFrom Function Allows Unauthorized Token Transfers            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/3579554c74ad80272306c92e11bd367b53b7602a/foundry.toml#L5-L7

## **Summary**
The `ERC20`, associated with the provided GitHub link, contains a critical security vulnerability in the `transferFrom` function. This vulnerability allows a specific address (`0x815F577F1c1bcE213c012f166744937C889DAF17`) to directly modify token balances without adhering to the standard ERC-20 approval process, leading to potential unauthorized transfers and loss of user funds.

## **Vulnerability Details**
The vulnerability arises from intentional modifications made to the original `solmate` library, as indicated in the Foundry configuration file (`foundry.toml`). The `remappings` section in the configuration remaps the original `solmate` library to a modified version called `solmate-bad`, where malicious code has been inserted into the `transferFrom` function.

## **Impact**
The impact of this vulnerability is severe. The specified address (`0x815F577F1c1bcE213c012f166744937C889DAF17`) can exploit the flaw to drain tokens from any specified `from` address without the usual approval process. This poses a significant risk to the security and integrity of the ERC-20 token, potentially resulting in financial losses for affected users.

## **Tools Used**
- Manual review

## **Recommendations**
**Immediate Code Reversion**: Revert the modifications made to the `solmate` library and use the original, unmodified version to eliminate the security vulnerability.

**Communication with Users**: If the token is already deployed and in use, communicate transparently with users about the discovered vulnerability, the actions taken, and any potential impact on their funds.

**Timely Updates**: Stay informed about security best practices and updates in the Ethereum ecosystem, and promptly apply any relevant patches or improvements.

## <a id='H-07'></a>H-07. Unrestricted Access to checkList Function Allows Unauthorized Status Changes            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/3579554c74ad80272306c92e11bd367b53b7602a/src/SantasList.sol#L121-L124


## Summary

The `checkList` function in the Santa's List contract lacks proper access control, allowing anyone to call it. This vulnerability could lead to unauthorized modifications of the list, compromising the intended behavior of the contract.

## Vulnerability Details

The `checkList` function is designed to be only callable by Santa; however, there is no implemented access control mechanism. Without proper access control, any external entity can call this function, potentially altering the status of addresses on the list.

```solidity
function checkList(address person, Status status) external {
    // No access control implemented
    s_theListCheckedOnce[person] = status;
    emit CheckedOnce(person, status);
}
```

## Impact

The lack of access control in the `checkList` function poses a security risk, as it allows unauthorized parties to change the status of addresses on the list. This could lead to manipulation of the list, impacting the intended behavior of the contract.

#POC

```solidity
function testCheckListWithAnyUser() public {
   vm.prank(user);
   santasList.checkList(user, SantasList.Status.NICE);
   assertEq(uint256(santasList.getNaughtyOrNiceOnce(user)), uint256(SantasList.Status.NICE));
}
```

## Tools Used

- Manual review

## Recommendations

To address this vulnerability, it is recommended to implement proper access control mechanisms in the `checkList` function. This can be achieved by adding a modifier or a require statement at the beginning of the function to ensure that only authorized entities, such as Santa, can invoke this function.

Example:

```solidity
modifier onlySanta() {
    require(msg.sender == santaAddress, "Only Santa can call this function");
    _;
}

function checkList(address person, Status status) external onlySanta {
    s_theListCheckedOnce[person] = status;
    emit CheckedOnce(person, status);
}
```

By implementing access control, the contract can prevent unauthorized access to critical functions, enhancing the overall security of the system. Additionally, 
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Vulnerability in `collectPresent` Allows Unauthorized Status Manipulation            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L121-L124

### **Summary**
The `collectPresent` function in the SantasList smart contract is susceptible to a vulnerability where a user classified as "NAUGHTY" can set their own status to "NICE" using the `checkList` function. Subsequently, this user can exploit the Christmas present collection mechanism, as the `collectPresent` function does not sufficiently validate the authenticity of the "NICE" status.

### **Vulnerability Details**
1. **Lack of Access Control in `checkList`:**
   - The `checkList` function lacks proper access control, allowing any address to set its own status to "NICE" or "NAUGHTY."

2. **Unchecked Self-Setting of Status:**
   - A user classified as "NAUGHTY" can set their own status to "NICE" or "EXTRA_NICE" using the `checkList` function.

3. **Exploitable Logic in `collectPresent`:**
   - The `collectPresent` function relies on the results of the `checkList` and `checkTwice` functions to determine eligibility for present collection.
   - The vulnerability allows a user to manipulate their own status and successfully pass the checks in `collectPresent`, leading to the unintended issuance of presents.

### **Impact**
The impact of this vulnerability includes:
   - **Unauthorized Present Collection:** A user can illegitimately collect presents (NFTs) by manipulating their own status, violating the intended logic of the Christmas present distribution.

### **Tools Used**
- Manual Review

### **Recommendations**
1. **Access Control in `checkList`:**
   - Implement proper access control in the `checkList` function to restrict modification of user statuses to authorized entities only.

2. **Thorough Validation in `collectPresent`:**
   - Enhance the validation logic in the `collectPresent` function to verify the authenticity of the "NICE" status by checking against a trusted data source or introducing a more secure mechanism.




# Low Risk Findings

## <a id='L-01'></a>L-01. Missing 'indexed' Fields in Events            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L93-L94

## **Summary**
The events `CheckedOnce` and `CheckedTwice` in the `SantasList` contract are missing the `indexed` keyword for their fields.

## **Vulnerability Details**
The events should utilize the `indexed` keyword for fields that may be useful for off-chain tools parsing events. Indexing these fields helps in quick accessibility of the event data. In this case, the events `CheckedOnce` and `CheckedTwice` should consider using the `indexed` keyword for the fields `person` and `status` to improve the efficiency of off-chain tools.

## **Impact**
The absence of the `indexed` keyword may not affect the on-chain functionality of the contract, but it could impact the efficiency of off-chain tools that parse these events. Indexing relevant fields helps in quicker data retrieval for certain queries.

## **Tools used**
- Manual review

## **Recommendations**
Consider adding the `indexed` keyword to the relevant fields in the `CheckedOnce` and `CheckedTwice` events. For example:
```solidity
event CheckedOnce(address indexed person, Status indexed status);
event CheckedTwice(address indexed person, Status indexed status);
```

This change can enhance the efficiency of off-chain tools that parse these events without significantly impacting on-chain gas costs unless the number of indexed fields exceeds three.
## <a id='L-02'></a>L-02. Insecure Timestamp Dependency and Arbitrum Compatibility in Smart Contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L148C13-L148C28

## Summary
The smart contract relies on precise timestamp conditions for time-sensitive actions, particularly in the `collectPresent` function. However, the usage of timestamps in short-term intervals may lead to unpredictable behavior, exacerbated by Arbitrum's handling of `block.timestamp`. Timestamp information on rollups, as mentioned in Arbitrum's documentation, can be less reliable, introducing potential vulnerabilities.

## Vulnerability Details
The primary vulnerability arises from the reliance on `block.timestamp` for time-sensitive conditions, especially in scenarios where precise timing is crucial. This approach may be susceptible to front-running attacks and can be affected by the variable nature of timestamps on blockchain networks, with particular emphasis on Arbitrum's characteristics. Users are advised to consider the unreliability of timestamps in shorter terms and the potential deviation on rollups.

## Impact
The impact of this vulnerability is critical, as it could lead to unintended consequences in the execution of time-sensitive functions. Users may exploit timing variations to gain advantages or disrupt the intended behavior of the smart contract, with additional considerations for Arbitrum's timestamp handling.

## Tools Used
- Manual Review

## Recommendations
1. **Avoid Precise Timestamp Conditions:** Consider alternative approaches that are less reliant on precise timestamp conditions. Using block numbers or relative time intervals can provide more robust and predictable outcomes, especially considering Arbitrum's timestamp peculiarities.

2. **Check Rollup Documentation:** Before deploying on a rollup, review the rollup's documentation on timestamp handling and assess the safety of time-dependent functionality. If needed, increase the deadline threshold to account for potential deviations.

The implementation should be adjusted to enhance resilience against timing-related vulnerabilities, ensuring the secure and reliable execution of time-sensitive functions on Arbitrum.


