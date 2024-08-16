# Biconomy: Nexus - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. `computeAccountAddress` may not work on all EVM chains](#M-01)
    - ### [M-02. Nexus smart account is not in compliance with EIP-7579](#M-02)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: Biconomy

### Dates: Jul 8th, 2024 - Jul 15th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-biconomy)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 2
- Low: 0



    
# Medium Risk Findings

## <a id='M-01'></a>M-01. `computeAccountAddress` may not work on all EVM chains            



GitHub\
<https://github.com/Cyfrin/2024-07-biconomy/blob/9590f25cd63f7ad2c54feb618036984774f3879d/contracts/factory/K1ValidatorFactory.sol#L111>\
\
Summary
-------

The `computeAccountAddress` function calculates the expected address of a Nexus contract using the factory's deterministic deployment algorithm. It derives the deployment address from the EVM create instruction and returns it.

However, some chains, such as **ZkSync Era**, have different address derivation methods for `create` and `create2`, leading to incorrect address calculations by this function.

## Impact

The function may not work correctly on all **EVM-compatible chains**.

## Recommendation

Consider storing the account address directly to ensure compatibility across different EVM chains.

## <a id='M-02'></a>M-02. Nexus smart account is not in compliance with EIP-7579            



## Summary

[EIP-7579](https://eips.ethereum.org/EIPS/eip-7579#erc-165) clearly states that smart accounts **MUST** implement ERC-165. This includes returning false for any interface function that reverts instead of implementing the functionality. However, `Nexus.sol` does not implement ERC-165, making it non-compliant with EIP-7579.

## Impact

The lack of ERC-165 implementation in `Nexus.sol` results in non-compliance with EIP-7579. This non-compliance can lead to interoperability issues with other smart contracts & modules expecting ERC-165 support.

## Recommendation

Implement ERC-165 in `Nexus.sol` as recommended in the standard. This involves adding the `supportsInterface` function to check for supported interfaces & ensuring it returns `false` for unsupported interfaces.





