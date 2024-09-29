# ArkProject: NFT Bridge - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. `_collections` can lead to increased gas fees for future whitelisting operations](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Users cannot withdraw their tokens if the bridge is disabled](#M-01)
    - ### [M-02. No min gas limit check on L1->L2 deposits](#M-02)
    - ### [M-03. `UUPSOwnableProxied` is missing `initialize()` function](#M-03)
    - ### [M-04. Incorrect token transfer logic, `new_owners` not considered](#M-04)
    - ### [M-05. Bridge is not in compliance with ERC721 standard](#M-05)
    - ### [M-06. `depositTokens` can be front-run](#M-06)
    - ### [M-07. Unauthorized ownership claim due to incomplete renouncement in 2-step ownership transfer](#M-07)
    - ### [M-08. Re-deployment of same contract due to bug in Cairo version `<2.7`](#M-08)
- ## Low Risk Findings
    - ### [L-01. Reinitialization vulnerability in `UUPSOwnableProxied` contract: risk of unauthorized takeover](#L-01)
    - ### [L-02. Use of `CREATE` opcode is suspicious of reorg attack](#L-02)
    - ### [L-03. Potential front-running vulnerability in `initialize` function](#L-03)
    - ### [L-04. User tokens can get stuck in escrow](#L-04)
    - ### [L-05. Airdrop loss when token is in escrow](#L-05)
    - ### [L-06. Validation missing for `collectionL2` felt value in `setL1L2CollectionMapping` function](#L-06)
    - ### [L-07. Custom errors will not work in 0.8.0](#L-07)
    - ### [L-08. Critical functions are not emitting events](#L-08)
    - ### [L-09. Owner can set the base uri without informing users](#L-09)
    - ### [L-10. Users tokens can get lost if `ownerL2` is set to a contract address which doesn't support ERC721](#L-10)
    - ### [L-11. `_withdrawFromEscrow` return value is not checked during cancellation of request](#L-11)
    - ### [L-12. Insufficient verification during cancellation of request allows potential double spending](#L-12)
    - ### [L-13. `depositTokens` allow `ownerL2` to be zero address, which leads to locking of tokens in escrow](#L-13)
    - ### [L-14. No way to cancel L2->L1 failed messages, user can lose tokens permanently](#L-14)
    - ### [L-15. `setMessageCancellationDelay` and `addMessageHashesFromL2` can be exploited by malicious users](#L-15)
    - ### [L-16. Current design of collection ownership is highly dependent  on bridge admin](#L-16)
    - ### [L-17. No way for user to get refund](#L-17)
    - ### [L-18. Potential risk of asset lockup due to asymmetric collection whitelisting](#L-18)
    - ### [L-19. NFT holder can lose discord privileges during bridging.](#L-19)
    - ### [L-20. User nfts can be lost if `owner_l1` is set to a contract address which doesn't support ERC721](#L-20)
    - ### [L-21. `replace_class_syscall` does not run a constructor](#L-21)
    - ### [L-22. `set_l1_l2_collection_mapping` allows overriting, can lead to token loss](#L-22)
    - ### [L-23. Irretrievable NFT loss due to bridged token misplacement or compromise](#L-23)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: ArkProject

### Dates: Jul 31st, 2024 - Aug 28th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-ark-project)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 8
- Low: 23


# High Risk Findings

## <a id='H-01'></a>H-01. `_collections` can lead to increased gas fees for future whitelisting operations            



Github\
<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/Bridge.sol#L192>
------------------------------------------------------------------------------------------------------------------------------------------

<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/Bridge.sol#L352>

\
Summary
-------

If `_collections` grows unbounded, it could lead to high gas costs for operations that iterate through this array. This could impact the efficiency and cost of future whitelisting operations.

## Vulnerability Details

The `_whiteListCollection` function is responsible for managing the `_collections` array. This function is called during token withdrawals if the collection does not exist on L1, and also by the admin to enable or disable collections. Users bridging messages from L2 to L1 and admins adding entries can cause `_collections` to grow indefinitely. As the array expands, operations that iterate through it will become more expensive, resulting in higher gas costs for future whitelisting operations.

This issue becomes more risky when the global whitelist is disabled, because when global whitelist is disabled the protocol allow any collection tokens to be deposited.

## Impact

The growing size of `_collections` can lead to increased gas fees for whitelisting operations, affecting user costs. This inefficiency could also result in transaction failures or scalability issues.

## Recommendation

Regularly review and clean up the `_collections` array to ensure it does not grow excessively. Currently when disabling the collection it doesn't remove it from collections array. 

Use more efficient data structures or indexing methods to handle large collections.

Also, I think it would be better to add a check in the `withdrawTokens`to not call `_whiteListCollection`when global whitelist is disabled. Because when global whitelist is disabled the protocol allow any collection to deposit and withdraw, so this check should only be called when the global whitelist is enabled.

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Users cannot withdraw their tokens if the bridge is disabled            



## Summary

Users cannot withdraw their tokens  if the bridge is disabled.

## Vulnerability Details

The intended behavior, as confirmed by the sponsor, is that users should be able to withdraw their tokens regardless of the bridge's enabled or disabled status. However, the current implementation incorrectly restricts token withdrawals when the bridge is disabled, contradicting this intended behavior.

## Impact

Users are unable to withdraw their tokens when the bridge is disabled, which is inconsistent with the expected behavior.

## Recommendation

1. **Remove the Bridge Status Check**: If you check cancellation functions they don't have the bridge enable/disable check so similarly modify the `withdrawTokens` function to remove the check for the bridge's enabled/disabled status. This will ensure that users can withdraw their tokens regardless of the bridge's status.
2. **Apply Fix to Cairo Bridge Contract**: Ensure that the same fix is applied to the Cairo bridge contract to maintain consistency across implementations.
3. **Test Changes**: Thoroughly test the updated implementation to confirm that the withdrawal process works as intended in all scenarios.

## <a id='M-02'></a>M-02. No min gas limit check on L1->L2 deposits            



## Github
https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/Bridge.sol#L137-L141

## Summary

The current bridging implementation from L1 to L2 does not include a minimum gas limit check. This oversight can result in tokens being stuck in escrow if the gas provided is below the required minimum threshold.

## Vulnerability Details

In the `sendMessageToL2` function, which facilitates messaging from L1 to L2, there is a crucial requirement for a minimum gas limit. According to the official [Cairo Docs](https://book.cairo-lang.org/ch16-04-L1-L2-messaging.html?highlight=msg.value#sending-messages-from-ethereum-to-starknet:~:text=It's%20important%20to%20note%20that%20we,be%20deserialized%20and%20processed%20on%20L2), the `msg.value` should be at least 20k wei. This minimum is necessary because:

* The StarknetMessaging contract needs to register the hash of the message in ETH storage.
* In addition to the 20k wei, sufficient fees must be paid on L1 to cover the deserialization and processing of the message on L2.

Without a minimum gas check, if the gas provided is below this threshold, the message may fail, causing the tokens to be stuck in escrow. Of-course the message can be retired later by cancelling but the issue still exist.

## Impact

If the minimum gas requirement is not met, user tokens can become stuck in escrow, potentially leading to significant issues for users and affecting the overall reliability of the bridging process.

## Recommendation

Implement a minimum gas limit check in the `sendMessageToL2` function to ensure that the gas provided meets the required threshold. This check will prevent tokens from being stuck in escrow due to insufficient gas and enhance the robustness of the bridging process.


## <a id='M-03'></a>M-03. `UUPSOwnableProxied` is missing `initialize()` function            



## Github

<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/UUPSProxied.sol#L14>

## Summary

The `UUPSOwnableProxied` contract inherits from OpenZeppelin's `Ownable` and `UUPSUpgradeable` contracts, intending to provide a convenient ownable UUPS proxy. However, the contract lacks an initializer function to set the initial owner, leaving the contract without a defined owner upon deployment.

## Impact

Without an initial owner set:

1. Functions protected by the `onlyOwner` modifier cannot be executed by any user, rendering these functions unusable.
2. The contract cannot be properly managed or upgraded as intended, leading to potential security & operational risks.

## Proof of Concept

* Deploy the `UUPSOwnableProxied` contract.
* Attempt to call any `onlyOwner` function ( e.g., `_authorizeUpgrade` ).
* The function will revert since no owner is set.

## Recommendation

Implement an initializer function to set the initial owner during the first deployment. This ensures that the owner is correctly set and the `onlyOwner` functions can be used as intended.

New code should look like this:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "openzeppelin-contracts/contracts/access/Ownable.sol";
import "openzeppelin-contracts/contracts/proxy/utils/UUPSUpgradeable.sol";
import "openzeppelin-contracts-upgradeable/proxy/utils/Initializable.sol";

error NotSupportedError();
error NotPayableError();

/**
   @title Convenient contract to have ownable UUPS proxied contract.
*/
contract UUPSOwnableProxied is Initializable, Ownable, UUPSUpgradeable {

    // Mapping for implementations initialization.
    mapping(address => bool) private _initializedImpls;

    // onlyInit
    modifier onlyInit() {
        address impl = _getImplementation();
        require(!_initializedImpls[impl], "Already init");
        _initializedImpls[impl] = true;

        _;
    }

    /**
       @notice Initializes the contract setting the deployer as the initial owner.
       @param initialOwner The initial owner of the contract.
    */
    function initialize(address initialOwner) public initializer {
        _transferOwnership(initialOwner);
    }

    /**
       @notice Only owner should be able to upgrade.
    */
    function _authorizeUpgrade(address newImplementation) internal override onlyOwner { }

    /**
       @notice Ensures unsupported function is directly reverted.
    */
    fallback() external payable {
        revert NotSupportedError();
    }

    /**
       @notice Ensures no ether is received without a function call.
    */
    receive() external payable {
        revert NotPayableError();
    }
}
```

###

## <a id='M-04'></a>M-04. Incorrect token transfer logic, `new_owners` not considered            



Github\
\
<https://github.com/ArkProjectNFTs/bridge/blob/1bb58731d8e4c37a71d3611c8ea6163c9b019193/apps/blockchain/starknet/src/bridge.cairo#L128-L181>\
\
Summary
-------

In the FSM for Starklane in [Figma](https://www.figma.com/board/esIDAZS1UySOAtq7hMa5xQ/FSM-For-Starklane---L1---L2?node-id=0-40\&t=re3PdFrPGxovPRVn-4), it is specified that if `new_owners` is empty, the token should be transferred to `owner_L2`. Otherwise, it should be transferred to the corresponding index in `new_owners`. However, the current implementation does not check for `new_owners` and always transfers the token to `owner_L2`, which is not correct.

## Impact

Tokens will always be transferred to `owner_L2`, and `new_owners` will never receive tokens even if `new_owners` is not empty.

## Recommendation

Add logic to handle the `new_owners` field correctly.

e.g add below lines in `withdraw_auto_from_l1`:

```rust
let to = if req.new_owners.len() == 0 {
    req.owner_l2
} else {
    req.new_owners[i]
};
```

## <a id='M-05'></a>M-05. Bridge is not in compliance with ERC721 standard            



Github\
<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/starknet/src/token/erc721_bridgeable.cairo#L29>\
\
Summary
-------

The `erc721_bridgeable` contract is not in compliance with the ERC-721 standard due to missing implementation of the ERC-165 interface, which is a critical requirement for ERC-721 compliance.

## Vulnerability Details

According to the ERC-721 [specification](https://eips.ethereum.org/EIPS/eip-721#simple-summary), every ERC-721 compliant contract must implement both the ERC-721 and ERC-165 interfaces. ERC-165 is used to determine which interfaces a contract supports.

In the `erc721_bridgeable` contract, although the `SRC5 component` is declared and included, it is not utilized to register or verify the contract's support for specific interfaces. The contract's implementation omits the necessary steps to declare its compliance with ERC-165, which means it does not meet the ERC-721 standard requirements.

## Impact

The lack of ERC-165 compliance in the `erc721_bridgeable` contract could result in:

* **Interoperability Issues**: Other contracts or tools querying the supported interfaces may fail to recognize the contract as compliant with ERC-721, leading to integration and interaction failures.
* **Standard Compliance**: The contract's failure to adhere to ERC-165 undermines its adherence to the ERC-721 standard, which could affect its acceptance in marketplaces and other platforms that enforce standard compliance.

## Recommendation

To address this issue and ensure compliance with ERC-721 standards, the following actions are recommended:

1. **Implement ERC-165 Interface**: Ensure the `erc721_bridgeable` contract properly implements the ERC-165 interface. This involves defining the `supports_interface` method, which should be used to declare support for ERC-721 and other relevant interfaces.

2. **Register Interfaces Using SRC5**: According to the [OpenZeppelin documentation](https://docs.openzeppelin.com/contracts-cairo/0.11.0/introspection#:~:text=For%20a%20contract%20to%20declare%20its%20support%20for%20a%20given%20interface%2C%20we%20recommend%20using%20the%20SRC5%20component%20to%20register%20support%20upon%20contract%20deployment%20through%20a%20constructor%20either%20directly%20or%20indirectly), use the SRC5 component to register support for the interfaces the contract implements. This should be done in the constructor of the contract to ensure that interface support is declared upon deployment.

## <a id='M-06'></a>M-06. `depositTokens` can be front-run            



## Github

<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/Bridge.sol#L78-L144>

## Summary

The `depositTokens` function is vulnerable to front-running attacks, which can unfairly advantage users who execute their transactions before others. This issue arises from the FIFO (*First-In-First-Out*) nature of the sequencer on Layer 2 (L2), allowing transactions to be processed in the order they are received.

## Vulnerability Details

The current protocol does not address or mitigate the risk of front-running when `depositTokens` is called. In scenarios where incentives or rewards are offered based on transaction order, users can exploit the system by submitting transactions with the intent to front-run others. This results in the front-runner's transaction being processed before those of other users, leading to potential loss of incentives for the latter.

### &#x20;Examples cases:

1. **Arkproject Fee Refund Incentive:**
   Currently Arkproject is offering a fee refund for the first 1,000 tokens bridged to Starknet. A front-runner who submits their bridging transaction before others will receive the fee refund, while subsequent users who bridge afterward will miss out. This creates an unfair advantage and undermines the incentive structure intended to reward early participants.

2. **Incentives from other Collections:**
   Similar incentive structures from other collections could also be affected by front-running. e.g, if a collection provides rewards or bonuses for early bridging, a front-runner can capture these rewards, leaving legitimate participants without the promised benefits.

3. **Priority-Based Systems:**
   Any system that prioritizes transactions based on their order—such as granting access to exclusive features or early benefits—can be compromised by front-running. This allows individuals to bypass the intended distribution of benefits, disrupting fair access.

4. **Example with Bored Apes:**
   Gas wars are very comon in NFTs, let's consider a scenario where Bored Apes offers a \$2,000 reward for the first 1,000 bridges. Given that Bored Apes is a 10,000 NFT collection, the first 1,000 bridges are eligible for the reward. Front-running could lead to only those who execute their transactions first benefiting from this reward, disadvantaging others who bridge tokens later.

## Impact

Front-running can lead to a loss of incentives and rewards for users who are not able to execute their transactions first. This undermines the fairness of incentive distribution and can result in a significant disparity between front-runners and other participants.

## Recommendations

For fixing front-running issues there can be many solutions, but a solid solution is to implement a commit-reveal scheme, where users first submit a hashed commitment of their transaction and later reveal the details. This approach obscures transaction data until it is processed, reducing the opportunity for front-running.

## <a id='M-07'></a>M-07. Unauthorized ownership claim due to incomplete renouncement in 2-step ownership transfer            



## Summary

The current implementation of the `OwnableTwoStep` in `bridge.cairo`  and `erc721_bridgeable` allows a pending owner to accept ownership even after the original owner has renounced ownership. This issue arises because the `Ownable_pending_owner` state variable is not cleared when ownership is renounced, enabling the pending owner to claim ownership after the original owner believes the contract has been relinquished. The issue is also present in OZ Cairo contracts [v0.11.0](https://github.com/OpenZeppelin/cairo-contracts/blob/a83f36b23f1af6e160288962be4a2701c3ecbcda/src/access/ownable/ownable.cairo#L109-L134) implementation which is used by ArkProject.

## Impact

This vulnerability can lead to unauthorized ownership transfer, undermining the original owner's intent to leave the contract without an owner. It introduces a security risk where an unintended party (pending owner) can gain control of the contract after the original owner has renounced ownership, potentially leading to misuse or exploitation of the contract.

## Proof of Concept

1. The current owner calls `transfer_ownership`, setting `Ownable_pending_owner` to Bob.
2. Bob does not immediately accept ownership, leaving `Ownable_pending_owner` active.
3. The current owner calls `renounce_ownership`, believing they have relinquished control, setting the owner to the zero address.
4. Bob, as the pending owner, calls `accept_ownership` after the renouncement.
5. Bob becomes the new owner of the contract, despite the original owner's intent to leave the contract without an owner.

## Recommendation

To address this issue, I think you need to override the OZ  `renounce_ownership` to ensure that `Ownable_pending_owner` is cleared (*set to zero address*) whenever `renounce_ownership` is called. This would prevent any pending owner from accepting ownership after the original owner has renounced it. 

Also, please note that the issue has been confirmed by the sponsor in a private thread and will be reported to OpenZeppelin also after the contest ends.

## <a id='M-08'></a>M-08. Re-deployment of same contract due to bug in Cairo version `<2.7`            



## Summary

A [vulnerability](https://github.com/starkware-libs/cairo/issues/5361) exists in Cairo version `<2.7`, specifically in version `2.6.3` currently used by the ArkProject. The `deploy_syscall` function in this version does not throw an error when the same contract is redeployed. This issue has been [resolved](https://github.com/starkware-libs/cairo/pull/5365/commits/8060cdad74bcce52c938c1a3da9479e2ecd50ccd) in Cairo `v2.7.0`, but it remains a concern in the current environment.

## Vulnerability Details

In Cairo version `2.6.3`, the `deploy_syscall` function permits the re-deployment of the same contract multiple times without raising an error. Within the ArkProject, the function `set_l1_l2_collection_mapping` allows overwriting existing mappings and even setting collections to a zero address for removal:

```rust
fn set_l1_l2_collection_mapping(ref self: ContractState, collection_l1: EthAddress, collection_l2: ContractAddress) {
  ensure_is_admin(@self);
  self.l1_to_l2_addresses.write(collection_l1, collection_l2);
  self.l2_to_l1_addresses.write(collection_l2, collection_l1);
}
```

If a mapping is removed and someone bridges the removed collection tokens from L1->L2 and `withdraw_auto_from_l1` is invoked by sequencer, which subsequently calls `deploy_erc721_bridgeable`, it will permit redeployment of a contract already deployed. This behavior, caused by the bug in Cairo `2.6.3`, allows for unintended redeployment on StarkNet, which is logically incorrect.

## Impact

The issue in Cairo `2.6.3` allows multiple redeployments of the same contract, the redeployment issue in ArkProject could allow a malicious user to exploit the system. If an admin removes a collection mapping by setting it to zero due to violations by the collection, they might believe that it is permanently removed and cannot be redeployed on StarkNet. However, the malicious user could still redeploy the same contract when bridging from L1 to L2. This occurs because the sequencer calls `withdraw_auto_from_l1`, which triggers the redeployment by invoking `deploy_erc721_bridgeable`. This could lead to unintended redeployment of the malicious collection contract, undermining the admin's actions and potentially causing security and operational issues within ArkProject.

## Recommendation

Update the project to the latest version of Cairo, preferably `v2.7.0` or higher, where this issue has been resolved.


# Low Risk Findings

## <a id='L-01'></a>L-01. Reinitialization vulnerability in `UUPSOwnableProxied` contract: risk of unauthorized takeover            



Github\
<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/UUPSProxied.sol#L14>\
\
Summary
-------

The `UUPSOwnableProxied` contract's functionality can be rendered inoperable, potentially halting the project, due to its vulnerability to reinitialization attacks.

## Vulnerability Details

According to OpenZeppelin's docs, leaving a contract uninitialized poses a significant security risk. An uninitialized contract can be taken over by an attacker. This risk applies to both the proxy and its implementation contract. If the implementation contract is not locked, it remains vulnerable to unauthorized reinitialization. To prevent such attacks, it is recommended to invoke the `_disableInitializers()` function in the constructor. This function ensures that the implementation contract is locked and cannot be initialized again after deployment.

## Impact

An attacker could take over the uninitialized contract, changing the owner and gaining control over the contract's state and upgrade process. This could lead to the attacker rendering the contract inoperable or maliciously altering its behavior, effectively halting the project.

## Recommendation

Invoke the `_disableInitializers()` function in the contract's constructor, as recommended by OpenZeppelin, to lock the implementation contract and prevent unauthorized reinitialization attacks. This step is crucial to secure the contract against takeover attempts and ensure its continued, secure operation.

## <a id='L-02'></a>L-02. Use of `CREATE` opcode is suspicious of reorg attack            



Github\
\
<https://github.com/Cyfrin/2024-07-ark-project/blob/a1e10ca451f9e24d33b4418009eece666f14f15e/apps/blockchain/ethereum/src/token/Deployer.sol#L30>\
\
Summary
-------

The `deployERC721Bridgeable` function deploys a UUPS proxied ERC721 contract using the CREATE opcode, which is vulnerable to reorg attacks. The standard `CREATE` method relies on the nonce of the transaction sender (*the Deployer library contract in this case*), making it risky in the event of a reorg.

## Vulnerability Details

The ArkProject NFT Bridge allows users to transfer Everai NFTs between Ethereum (L1) & Starknet (L2). It relies on the Starknet messaging protocol for this bridging process. The key components include the bridge admin who manages the bridge settings and users who perform the bridging of NFTs.

When `withdrawTokens` is called, it allows users to withdraw NFTs from L2 to L1. This function calls `_deployERC721Bridgeable`, which deploys a new ERC721Bridgeable contract on L1 if the corresponding contract doesn't exist. `_deployERC721Bridgeable` then calls `deployERC721Bridgeable`, which uses the CREATE opcode to deploy the proxy contracts. The CREATE opcode is problematic because it is vulnerable to reorgs.

ArkProject will be deployed only on **Ethereum and Starknet**, and reorgs can occur on these L1 and L2 chains respectively. E.g, here is a previous reorg that happened on Ethereum: [Ethereum reorg example](https://decrypt.co/101390/ethereum-beacon-chain-blockchain-reorg) - 2 years ago.

The vulnerability here is that users rely on address derivation in advance or when trying to deploy the same address on different chains. Any funds requested for the withdrawal can be stolen due to reorg manipulation.

## Impact

A malicious actor can exploit reorg events to deploy a malicious contract at the address intended for legitimate use, enabling them to steal NFTs during the withdrawal process from Starknet (L2) to Ethereum (L1).

## Proof of Concept

1. Alice, a legitimate user, wants to withdraw her Everai NFT from Starknet (L2) to Ethereum (L1).
2. The bridge admin has enabled the bridge and the whitelist for the Everai NFT collection.
3. Bob, a malicious actor, sets up a bot that monitors the Ethereum blockchain for reorg events.
4. Bob knows that during a reorg, the nonce of the deploying contract can change, affecting the address generated by the CREATE opcode.
5. Alice initiates the `withdrawTokens` function to withdraw her NFT.
6. The contract checks the message, and since the corresponding L1 contract doesn't exist, it prepares to deploy a new ERC721Bridgeable contract using CREATE.
7. Just as Alice's transaction is about to be mined, a reorg occurs.
8. Bob's bot detects the reorg and quickly deploys his own malicious ERC721Bridgeable contract to the address that the ArkProject NFT Bridge contract would use post-reorg.
9. Bob’s contract mimics the interface of the legitimate ERC721Bridgeable contract but includes a malicious backdoor.
10. The reorg completes, and Alice's transaction is mined. Due to the reorg, the nonce has changed, and the contract address generated by Alice's transaction now points to Bob's malicious contract.
11. Alice’s transaction finalizes, unknowingly interacting with Bob’s contract.
12. Alice's NFT is transferred to Bob’s malicious contract.
13. Bob’s contract, using the backdoor, transfers the NFT to Bob’s address.
14. Bob successfully steals Alice’s NFT.
15. Alice is unaware that her NFT has been diverted to a malicious contract due to the reorg and nonce manipulation.

## Recommendation

To prevent this attack, the bridge should use `CREATE2` with a salt to ensure that the contract address is deterministic and not dependent on the nonce, which can change during reorgs.&#x20;

### This is how it should be implemented with `CREATE2`

Modify the `deployERC721Bridgeable` function to use `CREATE2`:

```solidity
function deployERC721Bridgeable(
    string memory name,
    string memory symbol,
    bytes32 salt
)
    public
    returns (address)
{
    address impl = address(new ERC721Bridgeable());

    bytes memory dataInit = abi.encodeWithSelector(
        ERC721Bridgeable.initialize.selector,
        abi.encode(name, symbol)
    );

    address proxy;
    bytes memory bytecode = abi.encodePacked(
        type(ERC1967Proxy).creationCode,
        abi.encode(impl, dataInit)
    );

    assembly {
        proxy := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
        if iszero(proxy) {
            revert(0, 0)
        }
    }

    return proxy;
}
```

Update `_deployERC721Bridgeable` to pass the required salt:

```solidity
function _deployERC721Bridgeable(
    string memory name,
    string memory symbol,
    snaddress collectionL2,
    uint256 reqHash
)
    internal
    returns (address)
{
    bytes32 salt = keccak256(abi.encodePacked(msg.sender, reqHash));
    address proxy = Deployer.deployERC721Bridgeable(name, symbol, salt);
    _l1ToL2Addresses[proxy] = collectionL2;
    _l2ToL1Addresses[collectionL2] = proxy;

    emit CollectionDeployedFromL2(
        reqHash,
        block.timestamp,
        proxy,
        snaddress.unwrap(collectionL2)
    );

    return proxy;
}
```

This approach will enhance the security of the ArkProject NFT Bridge by mitigating the risk of reorg attacks.

## <a id='L-03'></a>L-03. Potential front-running vulnerability in `initialize` function            



## Github

<https://github.com/Cyfrin/2024-07-ark-project/blob/a1e10ca451f9e24d33b4418009eece666f14f15e/apps/blockchain/ethereum/src/Bridge.sol#L44-L66>

## Summary

The `UUPSOwnableProxied` contract includes an `onlyInit` modifier to ensure the `initialize` function is called only once. However, the `initialize` function in `Bridge.sol` can be front-run due to the deployment scripts not performing all actions in a single transaction. This allows any user to potentially manipulate the initialization process and claim administrative control over the contract.

## Vulnerability Details

The  Bridge involves deploying and initializing proxy contracts for bridging NFTs (ERC-721) between Ethereum (L1) and Starknet (L2). The key components include the bridge admin who manages the bridge settings and users who perform the bridging of NFTs.

**Initialization Process:**

* The `initialize` function in `Bridge.sol` is protected by the `onlyInit` modifier from the `UUPSOwnableProxied` contract to ensure it is only called once.
* This function sets critical parameters, including the owner, Starknet core address, and other necessary configurations.

**Deployment Scripts:**

* The provided deployment scripts perform contract deployment and initialization in separate transactions.
* This gap between deployment and initialization can be exploited by a malicious actor who can front-run the initialization transaction.

## Impact

* If a malicious actor front-runs the initialization process, they can call the `initialize` function and set themselves as the owner.
* This would give them control over the contract, allowing them to manipulate the bridge settings, including enabling/disabling the bridge, setting mappings, and potentially stealing NFTs or funds.

## Proof of Concept

1. Alice, the legitimate bridge admin, deploys the ArkProject NFT Bridge contract using the provided scripts.

2. The contract is deployed but not yet initialized.

3. Bob, a malicious actor, monitors the blockchain for new contract deployments. Upon detecting the deployment of the Bridge contract, Bob quickly sends a transaction to call the `initialize` function.

4. Bob's transaction gets mined before Alice's initialization transaction.

5. Bob sets himself as the owner during the initialization process.

6. With administrative control, Bob can disable the bridge, whitelist malicious contracts, and potentially steal any NFTs or funds bridged through the contract.

7. Alice loses control over the bridge. Users bridging NFTs can have their assets stolen or manipulated.

## Recommendations

To prevent this attack, the deployment and initialization process should be atomic, ensuring both actions occur within the same transaction. This can be achieved by updating the deployment scripts to perform both actions together.


## <a id='L-04'></a>L-04. User tokens can get stuck in escrow            



## Github

<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/Bridge.sol#L368-L375>

<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/token/CollectionManager.sol#L111-L134>

<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/Bridge.sol#L153-L215>

## Summary

User tokens can get stuck in escrow forever if `setL1L2CollectionMapping` is called and it updates `_l2ToL1Addresses`.

## Vulnerability Details

`setL1L2CollectionMapping` is an admin-controlled function that updates `_l1ToL2Addresses` and `_l2ToL1Addresses` mappings in `CollectionManager`. The problem with this function is that it can overwrite already saved collections due to the `force` parameter in `_setL1L2AddressMapping`. This overwrite can lead to a critical issue where user tokens get stuck in escrow forever.

When the `_l1ToL2Addresses` and `_l2ToL1Addresses` mappings are updated, it directly impacts the `withdrawTokens` function, which uses `_verifyRequestAddresses` for request verification. When an already used collection address is updated in `_l2ToL1Addresses`, the function `_verifyRequestAddresses` will return the new updated collection address, not the old/original one. If the new collection address is returned, the `withdrawTokens` function will mint a token from the new updated collection to the user instead of returning the original escrowed token.

```solidity
IERC721Bridgeable(collectionL1).mintFromBridge(req.ownerL1, id);
```

This results in the user's original token being stuck in escrow forever while they receive a new token from the updated collection.

## Impact

Although the `admin` is a trusted role, making the likelihood of this issue **low**, the impact is very **high**. User tokens can get stuck in escrow forever due to a valid update by the admin, not necessarily a malicious action.

## Proof of Concept

Let's take an example scenario to understand the bug in detail:

1. Bob bridges his NFT Everai #1 and NFT Everai #2 from L1 to L2 by calling `depositTokens`. This action places Everai #1 and Everai #2 into the escrow contract on L1. New tokens are minted for Bob on Starknet (L2), representing his NFTs.
2. Bob decides to bridge back NFT Everai #1 to L1. Everai #1 is withdrawn from escrow and returned to L1, but Everai #2 remains in escrow.
3. The admin calls `setL1L2CollectionMapping` for some reason, updating/overwriting `_l2ToL1Addresses` from the original Everai address (*ABC*) to a new address (*XYZ*).
4. Bob attempts to bridge back Everai #2 from L2 to L1. He expects to receive the original Everai #2 from the original collection with the *ABC* address. However, due to the update, the `_l2ToL1Addresses` mapping now points to *XYZ*. As a result, Bob will receive an NFT from the new collection (*XYZ*), not the original (*ABC*).

## Recommendation

Overwriting the mappings should not be allowed in the first place. However, if it is necessary, there should be a check in `setL1L2CollectionMapping` to ensure that no tokens are in escrow for the collection address being updated. If there are tokens, they should be cleared first, and then the force update/overwrite should be done.

## <a id='L-05'></a>L-05. Airdrop loss when token is in escrow            



Github\
\
<https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/Escrow.sol>\
\
Summary
-------

Airdrops can be permanently lost if the user's token is in the escrow contract during the airdrop distribution.

## Vulnerability Details

User tokens can end up in the escrow contract in two ways:

1. **Bridging from L1 to L2**: When a user deposits their token into escrow to bridge from L1 to L2, the token is locked in escrow, and a new token is minted on L2.
2. **Failed Deposit Request**: When a user requests a deposit into escrow to bridge from L1 to L2, but the request fails, leaving the token in escrow. In this case, the user must wait **5 days** plus the team's response time to withdraw the token from escrow.

If the user's token is in escrow for either of these reasons and an airdrop is distributed to that collection, the airdrop will be sent to the escrow contract instead of the user's account. Currently, there appears to be no way to withdraw the airdrop from the escrow contract.

## Impact

Users may permanently lose airdrops if their tokens are in escrow.

## Proof of Concept

Consider a scenario where a \`20k$\` airdrop is scheduled for a specific time or day for a collection, and the protocol distributes it to all holders. If the user's NFT is in escrow, the airdrop would go to the escrow contract. Since there is no mechanism to withdraw tokens from escrow, the airdrop funds/tokens will be stuck in escrow permanently.

## Recommendation

* Implement a solution that allows users to receive airdrops even if their tokens are in escrow.
* Alternatively, add a restricted function that enables users to withdraw stuck airdrops from the escrow contract.

## <a id='L-06'></a>L-06. Validation missing for `collectionL2` felt value in `setL1L2CollectionMapping` function            



Github\
\
<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/Bridge.sol#L368-L375>\
\
Summary
-------

The `setL1L2CollectionMapping` function in the Bridge.sol contract does not check if the `collectionL2` parameter, of type `snaddress`, is a valid felt value. This could potentially lead to issues if an invalid value is entered, even though the function is restricted to the contract owner. Similar issues in the previous audit is marked `medium`but i am submitting this one as `low`because the function is restricted so likelihood is low but the impact can be high.

## Impact

Even though the function is restricted to the admin (*trusted role*), there is a risk that an invalid felt value could be entered for `collectionL2`, leading to potential operational issues or failures in the bridge functionality.

## Proof of concept

The current implementation of the function does not validate that `collectionL2` is a valid felt value:

```solidity
function setL1L2CollectionMapping(
    address collectionL1,
    snaddress collectionL2,
    bool force
) external onlyOwner {
    _setL1L2AddressMapping(collectionL1, collectionL2, force); 
    emit L1L2CollectionMappingUpdated(collectionL1, snaddress.unwrap(collectionL2));
}
```

Without validation, if `collectionL2` is not within the valid range for felt values, it can cause issues when interacting with StarkNet.

## Recommendation

Add a validation check to ensure `collectionL2` is within the valid range for felt values.&#x20;

```solidity
function setL1L2CollectionMapping(
    address collectionL1,
    snaddress collectionL2,
    bool force
) external onlyOwner {
    require(Cairo.isFelt252(snaddress.unwrap(collectionL2)), "Invalid felt252 value for collectionL2");
    _setL1L2AddressMapping(collectionL1, collectionL2, force); 
    emit L1L2CollectionMappingUpdated(collectionL1, snaddress.unwrap(collectionL2));
}
```

By adding this validation, the function will ensure that `collectionL2` is always a valid felt value.

## <a id='L-07'></a>L-07. Custom errors will not work in 0.8.0            



Github\
\
<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/token/TokenUtil.sol#L3>\
\
Summary
-------

Custom errors were introduced in Solidity version `0.8.4`. However, the specified Solidity version in the pragma statement is `0.8.0`, leading to a compilation error.

## Vulnerability Details

Custom errors were introduced in Solidity version `0.8.4`. This prevents smart contracts using version `0.8.0` from using this library.

Ref: [Solidity 0.8.4 Release Announcement](https://soliditylang.org/blog/2021/04/21/solidity-0.8.4-release-announcement)

## Impact

Smart contracts using version `0.8.0` of Solidity and trying to use custom errors will encounter a compilation error.

## Recommendation

Consider upgrading the Solidity version specified in the pragma statement to `0.8.4` or a later version that supports custom errors. This can be achieved by updating the pragma statement at the beginning of the contract:

```solidity
pragma solidity ^0.8.4;
```

## <a id='L-08'></a>L-08. Critical functions are not emitting events            



Github\
\
<https://github.com/ArkProjectNFTs/bridge/blob/1bb58731d8e4c37a71d3611c8ea6163c9b019193/apps/blockchain/starknet/src/bridge.cairo#L360-L364>\
\
Summary
-------

The functions `set_bridge_l1_addr`, `set_l1_l2_collection_mapping`, `set_erc721_class_hash`, and `enable_white_list` are sensitive setter functions in the bridge that modify critical state variables. These functions currently do not emit events, making it difficult for external observers or dApps to track and react to these changes. The absence of events reduces transparency and complicates the integration and monitoring of contract activity.

## Impact

Without events:

1. External observers cannot easily track changes to critical state variables.
2. dApps and other integrations may face difficulties in monitoring and responding to state changes in real-time.
3. The overall transparency of the contract's operations is reduced, potentially leading to trust issues and operational inefficiencies.

## Recommendation

To enhance transparency and facilitate better monitoring, incorporate appropriate event emissions within these setter functions. This will ensure that any changes to critical state variables are logged and can be easily tracked by external observers.

## <a id='L-09'></a>L-09. Owner can set the base uri without informing users            



## Github

<https://github.com/ArkProjectNFTs/bridge/blob/1bb58731d8e4c37a71d3611c8ea6163c9b019193/apps/blockchain/starknet/src/token/erc721_bridgeable.cairo#L164-L167>

## Summary

The `erc721_bridgeable.cairo` contract includes a function `set_base_uri` that allows setting the base URI for NFTs. The base URI is crucial as it references the images of tokens stored in decentralized storage, representing the core value of NFTs. Users typically pay to own these images, so the base URI should remain immutable to ensure users retain ownership of the images. However, the `set_base_uri` function permits the contract owner to modify this value, which could potentially alter the images owned by users.

## Impact

The function is admin controlled so the likelihood is **low** but the impact is **high**. Allowing the base URI to be changed by the contract owner poses a risk of altering the images associated with the NFTs. This can undermine the trust and value of the NFTs, as users may lose ownership of the original images they paid for. Such changes could lead to a loss of user confidence and the perceived integrity of the NFT collection.

## Recommendation

Evaluate whether the ability to update the URI is genuinely necessary for the collection. If deemed necessary, implement a transparent process to notify users well in advance before any changes to the URI are made. This ensures that users are aware and can take appropriate actions if needed. Additionally, consider implementing safeguards or governance mechanisms to prevent arbitrary changes and protect user interests.

## <a id='L-10'></a>L-10. Users tokens can get lost if `ownerL2` is set to a contract address which doesn't support ERC721            



Github\
\
<https://github.com/ArkProjectNFTs/bridge/blob/1bb58731d8e4c37a71d3611c8ea6163c9b019193/apps/blockchain/starknet/src/bridge.cairo#L161>\
\
<https://github.com/ArkProjectNFTs/bridge/blob/1bb58731d8e4c37a71d3611c8ea6163c9b019193/apps/blockchain/ethereum/src/Bridge.sol#L78-L144>\
\
Summary
-------

If `ownerL2` is set to a contract address which doesn't support ERC721, the tokens can be permanently lost.

## Vulnerability Details

The `transfer_from` function in the Cairo ERC721 implementation contains a warning stating: "This method may lead to the loss of tokens if `to` is not aware of the ERC721 protocol" (see [OpenZeppelin ERC721 Cairo implementation](https://github.com/OpenZeppelin/cairo-contracts/blob/eabfa029b7b681d9e83bf171f723081b07891016/packages/token/src/erc721/erc721.cairo#L166) and [this](https://docs.openzeppelin.com/contracts-cairo/0.11.0/erc721#:~:text=If%20using%20transfer_from%2C%20the%20caller%20is%20responsible%20to%20confirm%20that%20the%20recipient%20is%20capable%20of%20receiving%20NFTs%20or%20else%20they%20may%20be%20permanently%20lost.)). When bridging tokens back from L1 to L2, if `ownerL2` is set to a contract address that does not support ERC721 tokens, the `withdraw_auto_from_l1` function will attempt to transfer tokens to this contract address using the `transfer_from` function.

Since the recipient contract does not support ERC721 tokens, it will not be able to handle or manage these tokens properly. As a result, there will be no mechanism to retrieve or manage the tokens from the contract, effectively leading to their loss. This issue arises because the ERC721 protocol requires certain functions and handling mechanisms that the non-supporting contract will lack.

## Impact

Users' tokens can be lost if sent to a contract address that does not support ERC721. The tokens will be locked in the recipient contract, and users will not be aware of this issue until it is too late, resulting in potential significant financial losses.

## Recommendation

To mitigate this vulnerability:

1. **Implement Address Validation**: Before transferring tokens, validate that the `ownerL2` address supports the ERC721 protocol. This can be done by checking if the contract implements the ERC721 interface.
2. **User Warnings and Documentation**: Inform users through documentation and warnings about the risks of setting `ownerL2` to a contract address that does not support ERC721 tokens.
3. **Safe Transfer Mechanism**: Implement a safe transfer mechanism that reverts the transaction if the recipient contract does not support ERC721, ensuring that tokens are not lost.


## <a id='L-11'></a>L-11. `_withdrawFromEscrow` return value is not checked during cancellation of request            



Github\
\
<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/Bridge.sol#L264>\
\
Summary
-------

The return value of `_withdrawFromEscrow` is not checked when called in `_cancelRequest`. It might lead to silent failures during cancellation.

## Impact

Users might believe they have successfully canceled the request and withdrawn the token, but due to a silent failure, the token will not actually be withdrawn.

## Recommendation

Check the return value of `_withdrawFromEscrow` and revert if it is false.

```Solidity
function _cancelRequest(Request memory req) internal {
    uint256 header = felt252.unwrap(req.header);
    CollectionType ctype = Protocol.collectionTypeFromHeader(header);
    address collectionL1 = req.collectionL1; 
    for (uint256 i = 0; i < req.tokenIds.length; i++) {
        uint256 id = req.tokenIds[i];
        bool success = _withdrawFromEscrow(ctype, collectionL1, req.ownerL1, id); 
        require(success, "Token withdrawal from escrow failed");
    }
}

```

## <a id='L-12'></a>L-12. Insufficient verification during cancellation of request allows potential double spending            



Github\
\
<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/Bridge.sol#L223-L266>\
\
Summary
-------

The current cancellation flow in our bridge contract allows for potential exploitation due to inadequate verification checks. Specifically, the `startRequestCancellation` and `cancelRequest` functions only verify the existence of the deposit payload but do not check if the deposit has been **processed by StarkNet L2**. This oversight can be exploited to gain tokens on both L1 and L2 if the L2 fails to send a confirmation proof within the set timeframe.

## Impact

If the deposit payload is not confirmed by StarkNet L2 within the expected timeframe, an attacker could potentially exploit this by canceling a deposit that has already been processed. This would allow them to reclaim the tokens on L1 while still retaining the tokens on L2, effectively doubling their assets and leading to significant financial losses for the system.

## Proof of Concept

Let's discuss how this issue can arise and attacker/malicious user can exploit it to his advantage:

1. **Initiate a Deposit**: A user initiates a deposit from L1 to L2.
2. **Payload Exists but Not Confirmed**: The deposit payload exists on L1, but the confirmation proof from L2 is not received within the set timeframe.
3. **Cancel the Deposit**: The user calls `startRequestCancellation` and subsequently `cancelRequest` to cancel the deposit.
4. **Tokens on Both Sides**: The user successfully reclaims the tokens on L1, while the tokens remain on L2 due to the lack of confirmation proof, resulting in double spending.

## Recommendation

To prevent this exploitation, implement a mechanism to verify the status of the deposit on StarkNet L2. 

Also, ensure that a certain period has passed without receiving the confirmation proof before allowing cancellation.

Along with this, implement safeguards to prevent tokens from being claimed on both L1 and L2.

## <a id='L-13'></a>L-13. `depositTokens` allow `ownerL2` to be zero address, which leads to locking of tokens in escrow            



## Github  

https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/Bridge.sol#L78-L84

## Summary

The `depositTokens` function in the provided contract allows users to deposit tokens into escrow and initiates the transfer to L2. However, there is a potential issue when the `ownerL2` address is set to zero.

## Impact

If the `ownerL2` address is zero, it could lead to a situation where tokens are locked in escrow on L1 without being transferred to the intended recipient on L2. This could cause user funds to be inaccessible, leading to potential financial losses and a loss of trust in the system.

As it is user input so likelihood is very **low** and can be considred user mistake, but the impact is **high** because user tokens gets lock in the escrow. So the final severity will be **low**.

## Proof of Concept

The `depositTokens` function includes a parameter `ownerL2`, which represents the new owner's address on StarkNet. The function checks if `ownerL2` is a valid felt252 value, but it does not explicitly check if `ownerL2` is non-zero. Here’s a simplified version of the critical part of the function:

```solidity
function depositTokens(
    uint256 salt,
    address collectionL1,
    snaddress ownerL2,
    uint256[] calldata ids,
    bool useAutoBurn
)
    external
    payable
{
    if (!Cairo.isFelt252(snaddress.unwrap(ownerL2))) {
        revert CairoWrapError();
    }
    if (!_enabled) {
        revert BridgeNotEnabledError();
    }

    // Other checks and logic...

    req.ownerL2 = ownerL2;

    // Serialize and send the message to L2
    uint256[] memory payload = Protocol.requestSerialize(req);
    if (payload.length >= MAX_PAYLOAD_LENGTH) {
        revert TooManyTokensError();
    }

    IStarknetMessaging(_starknetCoreAddress).sendMessageToL2{value: msg.value}(
        snaddress.unwrap(_starklaneL2Address),
        felt252.unwrap(_starklaneL2Selector),
        payload
    );

    emit DepositRequestInitiated(req.hash, block.timestamp, payload);
}
```

## Recommendation

To mitigate this issue, add an explicit check to ensure that `ownerL2` is not zero. Here is a potential solution:

```solidity
function depositTokens(
    uint256 salt,
    address collectionL1,
    snaddress ownerL2,
    uint256[] calldata ids,
    bool useAutoBurn
)
    external
    payable
{
    if (!Cairo.isFelt252(snaddress.unwrap(ownerL2)) || snaddress.unwrap(ownerL2) == 0) {
        revert CairoWrapError();
    }
    if (!_enabled) {
        revert BridgeNotEnabledError();
    }

    // Other checks and logic...

    req.ownerL2 = ownerL2;

    // Serialize and send the message to L2
    uint256[] memory payload = Protocol.requestSerialize(req);
    if (payload.length >= MAX_PAYLOAD_LENGTH) {
        revert TooManyTokensError();
    }

    IStarknetMessaging(_starknetCoreAddress).sendMessageToL2{value: msg.value}(
        snaddress.unwrap(_starklaneL2Address),
        felt252.unwrap(_starklaneL2Selector),
        payload
    );

    emit DepositRequestInitiated(req.hash, block.timestamp, payload);
}
```

By ensuring `ownerL2` is non-zero, we can prevent tokens from being locked in escrow and ensure they are transferred to a valid address on StarkNet.

## <a id='L-14'></a>L-14. No way to cancel L2->L1 failed messages, user can lose tokens permanently            



## Github

https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/bridge.cairo

## Summary

**L2 -> L1 Messaging Cancellation:** There is no such mechanism currently available.

## Vulnerability Details

The official docs provides information about L1->L2 message [cancellation](https://docs.starknet.io/architecture-and-concepts/network-architecture/messaging-mechanism/#l2-l1_message_cancellation), but there is no corresponding mechanism for canceling a message when an error occurs in the L1 contract or any other issue that leads to tokens being stuck in escrow. This lack of a cancellation mechanism for L2->L1 requests can result in the permanent loss of tokens, as they remain stuck in the escrow contract.

## Impact

Users can permanently lose tokens if the message fails from L2->L1, as the tokens will remain stuck in the escrow contract.

## Proof of Concept

* Dana attempted to bridge his Bored Ape NFT #2 from L2 to L1.
* The request failed for some reason, resulting in Dana's Bored Ape NFT #2 being stuck in the escrow contract.
* Dana wants to cancel the request to withdraw the NFT from escrow, but there is no mechanism to do so.
* Consequently, Dana's Bored Ape NFT #2 is permanently stuck in the escrow contract.

## Recommendation

I propose two potential solutions to address this issue:

1. Implement a cancellation feature for L2->L1 messages, allowing users to cancel their requests if they fail.
2. Develop a method to withdraw tokens from escrow, enabling users to retrieve their tokens in case of an emergency.

These solutions would provide a way to handle failed transactions and prevent the permanent loss of tokens.

## <a id='L-15'></a>L-15. `setMessageCancellationDelay` and `addMessageHashesFromL2` can be exploited by malicious users            



## Summary

The `StarknetMessagingLocal` contract inherits from `StarknetMessaging` and includes functions such as `setMessageCancellationDelay` and `addMessageHashesFromL2` that lack proper access controls. These functions allow changes to the cancellation delay and the addition of L2 message hashes, which can be exploited if deployed in a production environment.

**Note**: I am aware that `StarknetMessagingLocal`is OOS but the functions in here impact the main protocol functionality so that's why reporting it.

## Impact

If the `StarknetMessagingLocal` contract is deployed in a live environment, a malicious actor could exploit the lack of access controls to manipulate critical aspects of the messaging system:

* **Infinite Cancellation Delay**: By setting the cancellation delay to an infinite value, the attacker could effectively disable the cancellation feature, leading to potential denial of service and financial losses.
* **Message Hash Flooding**: An attacker could flood the system with arbitrary message hashes using the `addMessageHashesFromL2` function, overwhelming the L2 system and potentially causing disruptions in the message processing pipeline.

These vulnerabilities could undermine the integrity and reliability of the Starknet messaging protocol.

## Recommendation

1. **Implement Access Controls**: Restrict access to the `setMessageCancellationDelay` and `addMessageHashesFromL2` functions to only authorized users or roles, such as administrators or contract owners.

2. **Use Conditional Compilation for Testing Functions**: Limit the availability of these functions to testing environments only. This can be achieved through conditional compilation or by deploying separate contracts for testing and production environments.


## <a id='L-16'></a>L-16. Current design of collection ownership is highly dependent  on bridge admin            



## Summary

The current bridge design for NFT collections on L1 and L2 chains places significant control in the hands of the bridge admin, particularly in transferring ownership of collections on the target chain (L2). This dependency on the bridge admin creates potential delays & centralization risks. Collection owners must rely on the bridge admin to transfer ownership before they can manage their assets on L2.

## Impact

* **Dependency on Bridge Admin:** Collection owners are dependent on the bridge admin for transferring ownership, which can cause delays and potential misuse of power.
* **Risk of Unauthorized Transfers:** The bridge admin has the authority to transfer ownership to any address, posing a risk if the admin's actions are not transparent or securely managed.
* **Lack of Autonomy for Collection Owners:** Owners cannot independently manage their collections on L2 without first obtaining ownership, limiting their control and flexibility.

## Proof of concept

* Let say Dandy is owner of Bored Ape Collection on L1 (ETH), he can manage all functionality of the collection on L2 e.g setting token uri, or base uri of the nfts
* Holders of Bored Ape Collection bridged to L2 for some reasons, during bridging new collection is created on L2. And on creation currently bridge is the owner of the Bored Ape Collection on L2 (Starknet)
* Now Dandy updated base uri or token uri on L1 for some reasons, but he can't update it on L2 instantly. For this he will have to request bridge admin to transfer ownership to him. This leads to temporary DoS here.
* Imagine a scenario where owner want to update the token uri for a specific event and he can't do it on L2 because he will have to request bridge admin to transfer ownership to him first then he can update it. 

## Recommendation

* **Implement a Claim Ownership Feature:** Introduce a feature that allows collection owners to claim ownership on L2 independently. This function should include a verification mechanism to ensure that only the rightful owner on L1 can claim ownership on L2.
* **Verification Process:** Use cryptographic proofs or signatures from the L1 contract to verify ownership before allowing the transfer on L2. This ensures that the process is secure and only legitimate owners can claim their collections.
* **Autonomous Management:** Allow the function to work in reverse (L2 to L1) as well, enabling owners to manage their collections across chains without relying on the bridge admin. This reduces centralization risks and improves the overall security and efficiency of the bridge.

## <a id='L-17'></a>L-17. No way for user to get refund            



## Summary

Normally, when a transfer fails in other bridges, there is an option to specify an address for gas refunds. The current bridge doesn't have any option for refund.

If you check the starknet [docs](https://docs.starknet.io/architecture-and-concepts/network-architecture/messaging-mechanism/#hashing_l2-l1:~:text=Sending%20an%20L2%20to%20L1%20message%20always%20incurs%20a%20fixed%20cost%20of%2020%2C000%20gas%2C%20because%20the%20hash%20of%20the%20message%20being%20sent%20must%20be%20written%20to%20L1%20storage%20in%20the%20Starknet%20Core%20Contract.) it says:

> Sending an L2 to L1 message always incurs a fixed cost of 20,000 gas, because the hash of the message being sent must be written to L1 storage in the Starknet Core Contract.

Now during bridging if user gas is more than the fixed cost of **20k** gas, the user will lose that and there is no way to get the refund

## Impact

There is no way of user getting refund, so he loses the fund if message fails.

## Recommendations

Introduce a refund address and functionality so that if the message request fails the user should get refund.

## <a id='L-18'></a>L-18. Potential risk of asset lockup due to asymmetric collection whitelisting            



## Summary

The current setup in the ArkProject bridge allows the bridge admin to independently whitelist or disable NFT collections on either L1 or L2 chains. This asymmetric management of whitelisting can lead to scenarios where users may unknowingly bridge valuable NFTs into a situation where they become trapped, particularly if a collection is disabled on L2 but remains active on L1.

## Impact

* **Risk of Asset Lock-In:** If a collection is disabled on L2 after an NFT has been bridged, the owner may not be able to bridge it back to L1, effectively trapping the asset in escrow.
* **User Trust Issues:** Users, unaware of the disabling, could bridge high-value NFTs to L2 only to find themselves unable to return the NFTs to L1, causing significant financial and emotional distress.
* **Operational Bottleneck:** The centralization of whitelisting power in the hands of the bridge admin without automatic synchronization between L1 and L2 whitelisting status increases the risk of operational errors and misuse of authority.

## Proof of concept

* Bob owns two valuable Bored Ape NFTs (#1 and #2) on the ETH (L1), each worth over 30 ETH.
* The bridge admin disables the Bored Ape collection on the L2 network, but Bob is unaware of this action.
* Motivated by an airdrop or other benefits on L2, Bob decides to bridge his Bored Ape NFT #1 from L1 to L2.
* The bridging process locks the NFT in **Escrow** on L1, and a corresponding token is minted for Bob on L2.
* After receiving the perks on L2, Bob tries to bridge his NFT back to L1 to withdraw it from escrow.
* Bob discovers that he cannot complete the bridging back process because the collection is disabled on L2.
* As a result, Bob's Bored Ape NFT #1 remains stuck in escrow, leaving him dependent on the bridge admin to re-enable the collection to retrieve his valuable asset.

## Recommendation

* **Synchronized Whitelisting:** Implement a policy where disabling a collection on one chain (L1 or L2) automatically triggers the disabling of the same collection on the other chain. This would prevent users from inadvertently bridging assets into a disabled state.
* **Pre-Bridge Whitelisting Check:** Introduce a verification step during the bridging process that checks the whitelisting status on both L1 and L2 before allowing the bridge transaction to proceed. If the collection is disabled on either chain, the user should be alerted and the transaction should be blocked.


## <a id='L-19'></a>L-19. NFT holder can lose discord privileges during bridging.            



## Summary

When assigning roles to NFT holders in Discord, the role is usually granted after verifying the NFT ownership in the user's wallet. This allows holders to access exclusive channels reserved for them. However, when an NFT holder bridges their token from L1 to L2, the token is moved into escrow contract from user wallet. As a result, the Discord bot may mistakenly identify this as the holder no longer possessing the NFT, leading to the automatic removal of the assigned role.

## Impact

This issue causes the user to lose access to private Discord channels, which are meant to be available only to verified NFT holders.&#x20;

## Recommendation

Implement a mechanism to recognize and account for NFTs that are in escrow due to bridging, ensuring that roles are not removed from users in these cases. Alternatively, update the verification process to accommodate NFTs that have been moved to L2, preserving the user's access to the exclusive channels.

## <a id='L-20'></a>L-20. User nfts can be lost if `owner_l1` is set to a contract address which doesn't support ERC721            



Github\
\
<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/ethereum/src/Bridge.sol#L208>\
\
Summary
-------

The current implementation of the `mintFromBridge` function in the bridge contract potentially allows ERC721 tokens to be sent to a contract address on L1 that does not support receiving ERC721 tokens. This issue arises because the function uses `_mint`, which does not verify if the receiving address can handle ERC721 tokens, leading to a potential loss of NFTs.

## Vulnerability details

When a user deposits NFTs on L2 for bridging to L1, they specify an `owner_l1` address, which is then used in the withdrawal process on L1. The `mintFromBridge` function mints the NFT directly to the `owner_l1` address without checking if the address can accept ERC721 tokens. If `owner_l1` is a contract address that does not support ERC721 tokens, the token might be permanently lost.

## Impact

The loss of valuable NFTs is possible if they are sent to a contract address on L1 that cannot handle ERC721 tokens, due to the use of `_mint` without proper checks. This could result in significant financial loss & user dissatisfaction.

## Proof of concept

1. A user deposits an NFT on L2 with a contract address as the `owner_l1`.
2. During withdrawal on L1, the `mintFromBridge` function mints the NFT to the `owner_l1` contract address.
3. The contract address does not support ERC721 tokens, resulting in the loss of the NFT.

## Recommendation

Replace the `_mint` function in `mintFromBridge` with a safer alternative that verifies the recipient's ability to handle ERC721 tokens, such as `safeMint`. This will ensure that NFTs are not sent to incompatible contract addresses, preventing potential loss of tokens.

## <a id='L-21'></a>L-21. `replace_class_syscall` does not run a constructor            



## Github

<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/starknet/src/bridge.cairo#L185-L197>

## Summary

The current upgrade mechanism in Cairo, as implemented in `bridge.cairo` and `erc721_bridgeable.cairo`, utilizes the `replace_class_syscall` function. This syscall does not invoke a constructor, which complicates the process of updating storage variables and may result in a temporary denial of service if storage updates are needed.

## Impact

Since `replace_class_syscall` does not run a constructor, updating storage variables directly in an upgrade process is not feasible, potentially leading to disruptions in contract functionality.

## Recommendation

I did some research into StarkNet forums and expert [discussions](https://youtu.be/9CIhHNrliW4?t=2086) confirms that this issue is a known limitation in Cairo upgrades. My suggestion to address storage updates, consider implementing a temporary class with an initialization function as a workaround to facilitate necessary storage changes during upgrades.

## <a id='L-22'></a>L-22. `set_l1_l2_collection_mapping` allows overriting, can lead to token loss            



Github\
<https://github.com/Cyfrin/2024-07-ark-project/blob/273b7b94986d3914d5ee737c99a59ec8728b1517/apps/blockchain/starknet/src/bridge.cairo#L360-L363>\
\
Summary
-------

The `set_l1_l2_collection_mapping` function in the bridge.cairo allows the admin to map an L1 collection address (`collection_l1`) to an L2 collection address (`collection_l2`). The function writes these mappings into two mappings: `l1_to_l2_addresses` and `l2_to_l1_addresses`. However, if this function is called multiple times with the same `collection_l1` or `collection_l2` values, the existing mappings will be overwritten without any checks, potentially leading to loss of critical mapping data.

## Impact

* **Overwriting of Existing Mappings:** The current implementation allows existing mappings to be overwritten. This can lead to the accidental loss of the previous association between L1 and L2 addresses, which could disrupt the bridge's operations and lead to incorrect behavior in subsequent interactions with these collections.

* **User tokens can get stuck:** User tokens can get stuck in escrow forever due to a valid mapping update by the admin, not necessarily a malicious action.

## Proof of concept

Let's consider an example scenario to demonstrate the potential issue with the `set_l1_l2_collection_mapping` function:

1. **Initial Deposit:** Bob bridges his NFT Everai #1 and NFT Everai #2 from L1 to L2 by calling `depositTokens`. This action places Everai #1 and Everai #2 into the escrow contract on L1. New tokens are minted for Bob on StarkNet (L2), representing his NFTs.

2. **Partial Withdrawal:** Bob decides to bridge back NFT Everai #1 to L1. Everai #1 is withdrawn from escrow and returned to L1, but Everai #2 remains in escrow.

3. **Admin Mapping Update:** The admin calls `set_l1_l2_collection_mapping` for some reason, updating/overwriting the `l1_to_l2_addresses` mapping from the original Everai address (*ABC*) to a new address (*XYZ*).

4. **Unexpected Outcome on Withdrawal:** Bob attempts to bridge back Everai #2 from L2 to L1. He expects to receive the original Everai #2 from the original collection with the *ABC* address. However, due to the update, the `l1_to_l2_addresses` mapping now points to *XYZ*. As a result, Bob will receive an NFT from the new collection (*XYZ*), not the original (*ABC*).

## Recommendation

The implementation should follow the L1-like approach by introducing a `force` option in the `set_l1_l2_collection_mapping` function. Overwriting the mappings should be disallowed by default. However, if an overwrite is necessary, the admin should explicitly specify the `force` option. Before allowing a force update or overwrite, the function should include a check to ensure that no tokens are currently in escrow for the collection address being updated. If tokens are found in escrow, they should be cleared first before proceeding with the force update or overwrite. This approach ensures that mappings are only overwritten when absolutely necessary and that any potential issues related to escrowed tokens are addressed before changes are made.

## <a id='L-23'></a>L-23. Irretrievable NFT loss due to bridged token misplacement or compromise            



## Summary

If a user bridges their NFT and loses the corresponding token on either L1 or L2, the original NFT can become permanently stuck in escrow.

## Impact

User tokens can be irretrievably stuck in escrow, leading to permanent loss of access to the NFT. The likelihood of this kinda situation is **low** but the impact is very **high**, so for that reason the severity is marked **medium**.

## Proof of Concept

* If an NFT like Bored Ape #7 is bridged from L1 to L2, the original NFT is placed in escrow, and a corresponding token #7 is issued on L2.
* If token #7 is lost on L2 due to hacking, burning, accidental listing on a marketplace (e.g., OpenSea), or being escrowed on another compromised platform, the original NFT in L1 escrow cannot be retrieved.
* A similar scenario applies when bridging from L2 to L1. Losing the token on either side results in the original escrowed NFT being stuck forever.

## Recommendation

I think there should be an emergency recovery mechanism to allow legitimate users to reclaim their tokens from escrow, ensuring that NFTs are not permanently lost due to issues on either side of the bridge.



