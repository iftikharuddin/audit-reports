# Checklist before auditing protocols

Did you ran slither? Aderyn? or any other static analysis tool? They can help a lot believe me!!

## Lending/Borrowing DeFi Attacks
 
Lending & Borrowing DeFi platforms display common sets of vulnerabilities

- No Incentive To Liquidate Small Positions e.g [Issue](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/issues/1096)
- Liquidation Before Default e.g [Issue](https://github.com/sherlock-audit/2023-03-teller-judging/issues/92)
- Borrower Can't Be Liquidated e.g [Issue](https://dacian.me/exploiting-developer-assumptions#heading-unchecked-return-values)
- Borrower Repayment Only Partially Credited e.g [Issue](https://github.com/sherlock-audit/2022-10-astaria-judging/issues/190)
- Debt Closed Without Repayment e.g [Issue](https://dacian.me/exploiting-developer-assumptions#heading-unexpected-empty-inputs)
- Repayments Paused While Liquidations Enabled e.g [Issue](https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/290)
- Token Disallow Stops Existing Repayment & Liquidation
- Borrower Immediately Liquidated After Repayments Resume
- Liquidator Takes Collateral With Insufficient Repayment
- Infinite Loan Rollover e.g [Issue](https://github.com/sherlock-audit/2023-01-cooler-judging/issues/215)
- Repayment Sent to Zero Address e.g [Issue](https://github.com/sherlock-audit/2023-01-cooler-judging/issues/33)
- Borrower Permanently Unable To Repay Loan e.g [Issue](https://github.com/sherlock-audit/2022-10-union-finance-judging/issues/133)
- Borrower Repayment Only Partially Credited e.g [Issue](https://github.com/sherlock-audit/2022-10-astaria-judging/issues/190)


Credits and thanks to: [Dacian](https://dacian.me/) & [mixbytes](https://mixbytes.io/blog/vulnerable-spots-of-lending-protocols)

## Oracles

- Assuming Oracle Price Precision [Stackoverflow](https://ethereum.stackexchange.com/questions/92508/do-all-chainlink-feeds-return-prices-with-8-decimals-of-precision)

- [M-01] The Chainlink price feed's input is not properly validated ( check for stale and negative returns )

### Chainlink Oracle Security Considerations

- Not Checking For Stale Prices
- Not Checking For Down L2 Sequencer e.g [Issue](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/issues/1046)
- Unhandled Oracle Revert Denial Of Service

### [](https://www.beirao.xyz/blog/Security-checklist#vault)Vault

[V-01] - Can transferring ERC20 or ETH directly break something ?

[V-02] - Is the vault balance tracked internally ?

[V-03] - Can the 1st deposit raise a problem ? [here](https://mixbytes.io/blog/overview-of-the-inflation-attack#rec558234839)

[V-04] - Is this vault taking into consideration that some ERC20 token are not 18 decimals ?

[V-05] - Is the fee calculation correct ?

[V-06] - What if only 1 wei remain in the pool ?

[V-07] - On vault with strategies implemented :

-   flash deposit-harvest-withdraw attacks are possible? First depositor can steal funds of others [here](https://mixbytes.io/blog/overview-of-the-inflation-attack#rec558234839)
-   How the vault behave when locked fund are put in a strategy?
-   Are losses handled ? (always should)
-   What happen in case of a black swan event ? (protocol implemented in the strategy get hacked)
-   Look at token-specific/protocol risks implemented in strategies :
    -   For protocol
        -   Pause ?
        -   Emergency withdrawal ?
        -   Depreciation ?
    -   For tokens
        -   All weird implementations

[V-08] - Can we manipulate the conversion rate between shares and underlying ? ([here](https://mixbytes.io/blog/yield-aggregators-common-pitfalls#rec515086866))

## Bridges DeFi Attacks

- 

ref https://github.com/ComposableSecurity/SCSVS/blob/master/2.0/0x200-Components/0x205-C5-Bridge.md 

## Cross Chain Attacks

- Verify that the protocol uses solidity-examples package to import LayerZero contracts and does not copy-paste them directly from the repository.

ref https://github.com/ComposableSecurity/SCSVS/blob/master/2.0/0x300-Integrations/0x304-I4-Cross-Chain.md 

## Extra known issues

- [L-02] It's possible to use a flawed compiler version
Solidity version 0.8.13 & 0.8.14 have a security vulnerability related to assembly blocks that write to memory.
The issue is fixed in version 0.8.15 and is explained [here](https://soliditylang.org/blog/2022/06/15/solidity-0.8.15-release-announcement/).
While the problem is with the legacy optimizer, it is still correct to enforce latest Solidity version (0.8.21) or at least the one enforced in other popular library code as OpenZeppelin (0.8.19).

- Reports: https://ottersec.notion.site/Sampled-Public-Audit-Reports-a296e98838aa4fdb8f3b192663400772

## IERC20 Issues

- Medium - Unsafe use of transfer() with IERC20
    - Recommendation: Use OpenZeppelin's SafeERC20 library instead of transfer.
    
## Informational issues    

- #I-1 : internal function name doesn't start with an '_'. They should start with _ ( best case )
    - Note: internal library function don't start with an `_`
    
- #-2: Hash Collision Vulnerability   
    - replace the usage of abi.encodePacked() with abi.encode()
    
- #-3: Events missing `indexed` fields for 
    - Index event fields make the field more quickly accessible to off-chain tools that parse events   
     