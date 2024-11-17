# Liquid Staking - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Operator can't withdraw the tokens during `removeOperators`](#H-01)

- ## Low Risk Findings
    - ### [L-01. No way to update unbonding and claim periods](#L-01)
    - ### [L-02. `updateStrategyRewards` is not called before adding & updating the fees](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Stakelink

### Dates: Sep 30th, 2024 - Oct 17th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-09-stakelink)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Operator can't withdraw the tokens during `removeOperators`            



## Github

<https://github.com/Cyfrin/2024-09-stakelink/blob/f5824f9ad67058b24a2c08494e51ddd7efdbb90b/contracts/linkStaking/OperatorStakingPool.sol#L163-L184>

<https://github.com/Cyfrin/2024-09-stakelink/blob/f5824f9ad67058b24a2c08494e51ddd7efdbb90b/contracts/linkStaking/OperatorStakingPool.sol#L199-L205>

## Summary

The `removeOperators` function in the OperatorStakingPool contract fails to actually transfer staked tokens back to operators when they are removed, despite updating internal accounting. This can lead to tokens being trapped in the contract and a mismatch between recorded and actual balances.

## Vulnerability details

When an operator is removed via the `removeOperators` function, it calls the `_withdraw` function to handle any staked tokens:

```solidity
function removeOperators(address[] calldata _operators) external onlyOwner {
    // ... (other code)
    uint256 staked = getOperatorStaked(operator);
    if (staked != 0) {
        _withdraw(operator, staked);
    }
    // ... (other code)
}
```

However, the `_withdraw` function only updates internal accounting without transferring tokens:

```solidity
function _withdraw(address _operator, uint256 _amount) private {
    uint256 sharesAmount = lst.getSharesByStake(_amount);
    shareBalances[_operator] -= sharesAmount;
    totalShares -= sharesAmount;

    emit Withdraw(_operator, _amount, sharesAmount);
}
```

This function updates share balances and emits a Withdraw event, but doesn't actually transfer any tokens to the operator.

## Impact

1. **Trapped Funds**: Staked tokens remain in the contract after an operator is removed, potentially becoming inaccessible.
2. **Accounting Discrepancy**: The contract's internal accounting no longer matches its actual token balance.
3. **Misleading Events**: Withdraw events are emitted without corresponding token transfers.
4. **Trust Issues**: Operators may lose trust in the system if they can't retrieve their staked tokens upon removal.

The severity is high due to the potential for permanent **loss of user funds** and the fundamental misalignment between the contract's stated behavior and its actual operation.

## What causes the issue and where?

The issue is caused by an incomplete implementation of the `_withdraw` function in the OperatorStakingPool contract. While it updates internal accounting, it fails to include the crucial step of transferring tokens back to the operator.

## Proof of concept

1. Admin adds an operator via `addOperators`.
2. Operator stakes 100 LST tokens.
3. Admin removes the operator via `removeOperators`.
4. The operator's internal balance is set to 0, and a Withdraw event is emitted.
5. However, the 100 LST tokens remain in the OperatorStakingPool contract.
6. The operator has no way to retrieve their staked tokens.

## Recommendations

Make sure there is a code which transfer the operator funds, here is two suggestions I have:

* Modify the `_withdraw` function to include an actual token transfer:

```solidity
function _withdraw(address _operator, uint256 _amount) private {
    uint256 sharesAmount = lst.getSharesByStake(_amount);
    shareBalances[_operator] -= sharesAmount;
    totalShares -= sharesAmount;

    lst.safeTransfer(_operator, _amount);  // Add this line

    emit Withdraw(_operator, _amount, sharesAmount);
}
```

* Add a separate function for operators to claim any remaining balance after removal, in case the automatic withdrawal fails:

```solidity
function claimRemainingBalance() external {
    uint256 balance = getOperatorStaked(msg.sender);
    require(balance > 0, "No balance to claim");
    require(!isOperator(msg.sender), "Active operators cannot claim");
    
    _withdraw(msg.sender, balance);
}
```

By implementing these recommendations, the contract will ensure that operators can always retrieve their staked tokens, maintaining the integrity and trustworthiness of the staking system.

    


# Low Risk Findings

## <a id='L-01'></a>L-01. No way to update unbonding and claim periods            



## Github

https://github.com/Cyfrin/2024-09-stakelink/blob/f5824f9ad67058b24a2c08494e51ddd7efdbb90b/contracts/linkStaking/FundFlowController.sol#L22-L25

## Summary

The **FundFlowController** contract has hardcoded values for unbonding and claim periods, while **Chainlink** can update these periods in their contracts via setters. This mismatch leads to discrepancies in timing, potentially causing issues with fund withdrawals. As Chainlink changes its periods, **FundFlowController** fails to stay in sync, resulting in delays or incorrect processing of user withdrawals.

## Where it occurs?

This issue arises in the **FundFlowController** contract, specifically in how it handles **unbonding and claim periods**. The contract relies on static time periods, which do not update dynamically when Chainlink modifies its own periods in related contracts.

```Solidity
// duration of the unbonding period in the Chainlink staking contract
uint64 public unbondingPeriod;
// duration of the claim period in the Chainlink staking contract
uint64 public claimPeriod;
```

## Actual Cause

The problem stems from **FundFlowController** using fixed, static time periods for unbonding and claiming, while **Chainlink** contracts have the flexibility to change these periods. The **FundFlowController** records the **start time** of the unbonding process but does not account for changes to the actual **end times** of unbonding and claim periods as set by Chainlink.

## Impact

If Chainlink modifies its unbonding or claim periods, the **FundFlowController** will operate based on outdated assumptions, leading to the following potential issues:

* **Withdrawal Delays**: Users can experience delays in accessing their funds if the actual periods shorten but the controller uses outdated timings.
* **Premature or Incorrect Withdrawals**: If the unbonding or claim periods are extended by Chainlink, withdrawals might be processed too early, resulting in failed transactions or reverted operations.
* **Locked Funds**: Users' funds may remain locked for longer than necessary, reducing liquidity and causing potential dissatisfaction with the system.

## Likelihood

The likelihood of this issue occurring is **low to moderate because** it is dependent on how frequently **Chainlink** modifies its periods. Given the evolving nature of Chainlink's contracts, there is a significant risk that these timing discrepancies will occur unless actively managed.

## Recommendations

1. **Dynamic Period Updates**: Modify the **FundFlowController** to dynamically fetch and synchronize the unbonding and claim periods from Chainlink's contracts, ensuring that the controller always operates based on the current period durations.

2. or better approach is to add setters for these values.


## <a id='L-02'></a>L-02. `updateStrategyRewards` is not called before adding & updating the fees            



Github\
\
<https://github.com/Cyfrin/2024-09-stakelink/blob/f5824f9ad67058b24a2c08494e51ddd7efdbb90b/contracts/core/StakingPool.sol#L347-L374>\
\
Summary
-------

In the Trust Security audit report, an issue titled **"TRST-L-4: Strategy rewards are not updated before updating the fees"** was identified in the **`SequencerVCS.sol`** contract. A **similar issue** is present in the **`StakingPool.sol`** contract. Specifically, the functions **`addFee()`** and **`updateFee()`** modify the fee structure without calling **`updateStrategyRewards()`** beforehand. This results in rewards being distributed based on **outdated fee values**, potentially leading to incorrect reward calculations.

## Impact

Failure to update the strategy rewards before modifying the fee structure can cause **incorrect reward distribution**. When the fees are changed without recalculating the rewards, the old reward values will not account for the newly updated fee structure. This could lead to overpayment or underpayment of rewards to stakeholders, depending on the timing of the fee change.

If unaddressed, this issue could:

1. **Distort reward calculations**, resulting in an inaccurate rewards distribution for stakeholders.
2. **Impact trust and fairness** in the staking system, as rewards may not reflect the correct deductions for fees.
3. **Potentially harm the staking pool's integrity**, as incorrect reward calculations can lead to stakeholder dissatisfaction and potential financial losses.

## What causes this issue?

The contract lacks a call to **`updateStrategyRewards()`** before modifying the fee structure in **`addFee()`** and **`updateFee()`**.

## Recommendations

**Call `updateStrategyRewards()`** before making any changes to the fee structure in both the **`addFee()`** and **`updateFee()`** functions. This will ensure that rewards are calculated and distributed based on the **old fee values** before they are modified.



