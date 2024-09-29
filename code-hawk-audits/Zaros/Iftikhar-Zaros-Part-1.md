# Zaros Part 1 - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. `priceFeedHeartbeatSeconds` is not added to the `PerpMarket` data](#M-01)
- ## Low Risk Findings
    - ### [L-01. Referrer overriding in `createCustomReferralCode` leads to misallocated rewards](#L-01)
    - ### [L-02. `Referral` and `CustomReferralConfiguration` libraries are not ERC7201 compliant](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Zaros

### Dates: Jul 17th, 2024 - Jul 31st, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-zaros)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1
- Low: 2



    
# Medium Risk Findings

## <a id='M-01'></a>M-01. `priceFeedHeartbeatSeconds` is not added to the `PerpMarket` data            



Github\
<https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/leaves/MarketConfiguration.sol#L32-L49>\
<https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/GlobalConfigurationBranch.sol#L379-L441>\
\
Summary
-------

The `createPerpMarket` function in the `GlobalConfigurationBranch` contract is responsible for creating and configuring a new perpetual futures market. It is a critical function that allows the owner of the contract to define the parameters of a new market, which includes various financial and operational settings. The creation flow is such. `GlobalConfigurationBranch:createPerpMarket` calls `PerpMarket.create` function which then calls `MarketConfiguration.update` with all the data sent as `params`.

## Vulnerability Details

Now the issue is `MarketConfiguration.update` is adding all other values to the `self` but missing one field and value which is `priceFeedHeartbeatSeconds`. This field is never added anywhere in the whole creation flow. The `priceFeedHeartbeatSeconds` is very crucial for getting updated and correct prices.

## POC

Add the following event to the `MarketConfiguration` contracts:

```Solidity
    event LogConfigurePriceFeedHeartbeatSeconds(uint32 priceFeedHeartbeatSeconds);
```

Then log this event in the `MarketConfiguration:update` function i.e:

```Solidity
    function update(Data storage self, Data memory params) internal {
        self.name = params.name;
        self.symbol = params.symbol;
        self.priceAdapter = params.priceAdapter;
        self.initialMarginRateX18 = params.initialMarginRateX18;
        self.maintenanceMarginRateX18 = params.maintenanceMarginRateX18;
        self.maxOpenInterest = params.maxOpenInterest;
        self.maxSkew = params.maxSkew;
        self.maxFundingVelocity = params.maxFundingVelocity;
        self.minTradeSizeX18 = params.minTradeSizeX18;
        self.skewScale = params.skewScale;
        self.orderFees = params.orderFees;
        emit LogConfigurePriceFeedHeartbeatSeconds(self.priceFeedHeartbeatSeconds);
    }
```

Now  run the following command to test it:

```Solidity
forge test --match-test test_WhenPriceFeedHearbeatSecondsIsNotZero -vvvv
```

It will emit a zero value for `priceFeedHeartbeatSeconds` variable and no matter what value is set in `test_WhenPriceFeedHearbeatSecondsIsNotZero` function, it will still be `0` . Which proves that the actual value is never added to the `self` variable during any step of the whole creation flow.

## Impact

The `priceFeedHeartbeatSeconds` parameter defines how often the price feed updates occur. With this parameter being `0`, it will always cause DOS wherever this price is required in the code. Accurate pricing is critical for the proper valuation of assets and for calculating margins, liquidations, and other financial metrics. If the price feed isn't updated responsive enough, it could lead to permanent DOS and potentially cause issues in margin calculations and trade executions.

## Recommendations

To resolve this issue, add the following line to the `MarketConfiguration:update` function:

```Solidity
self.priceFeedHeartbeatSeconds = params.priceFeedHeartbeatSeconds;
```


# Low Risk Findings

## <a id='L-01'></a>L-01. Referrer overriding in `createCustomReferralCode` leads to misallocated rewards            



Github\
<https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/GlobalConfigurationBranch.sol#L632-L636>\
\
Summary
-------

The `createCustomReferralCode` function allows `referrer` overriding.

## Impact

The new user will receive rewards for the previous user's activities if the referrer is overwritten.

## Proof of Concept

Assume User A refers User B using a custom referral code "REF123". User A expects to receive rewards for User B's activity. However, if admin overwrites the referrer for "REF123" to User C, User C will start receiving rewards instead of User A.

## Recommendation

Add a check to prevent overriding an existing referrer:

```solidity
function createCustomReferralCode(address referrer, string memory customReferralCode) external onlyOwner {
    CustomReferralConfiguration.Data storage config = CustomReferralConfiguration.load(customReferralCode);
    require(config.referrer == address(0), "Referral code already has a referrer");

    config.referrer = referrer;
    emit LogCreateCustomReferralCode(referrer, customReferralCode);
}
```

## <a id='L-02'></a>L-02. `Referral` and `CustomReferralConfiguration` libraries are not ERC7201 compliant            



GitHub\
<https://github.com/Cyfrin/2024-07-zaros/blob/main/src/perpetuals/leaves/Referral.sol>\
<https://github.com/Cyfrin/2024-07-zaros/blob/main/src/perpetuals/leaves/CustomReferralConfiguration.sol>\
\
Summary
-------

The `CustomReferralConfiguration` and `Referral` libraries are not [ERC7201](https://eips.ethereum.org/EIPS/eip-7201#specification) compliant. Neither library includes the required `@custom:storage-location` annotation in their structs. Additionally, both libraries use `keccak256` to compute the storage slot but do not follow the exact formula prescribed by ERC-7201 (`keccak256(abi.encode(uint256(keccak256(bytes(id))) - 1)) & ~bytes32(uint256(0xff))`).

## Impact

Storage collisions chances are high and also compatibility issues can occur, let me explain below:

* **Storage Collisions**: The ERC-7201 formula is designed to avoid collisions with the default storage layout used by Solidity and Vyper. Not following this formula increases the risk of storage collisions, where different variables or structs overwrite each other's storage slots. This can corrupt data and lead to unpredictable contract behavior.
* **Tooling Compatibility**: Blockchain explorers, debuggers, and other development tools may not recognize the storage layout without the annotation, leading to potential misinterpretations of storage data. This can complicate the debugging process and hinder the development workflow.

## Recommendation

* &#x20;Modify the slot computation to use the formula defined by ERC-7201.
* Add the NatSpec annotation to indicate the storage location.



