# Staking - Findings Report

**Note:** I did this contest as a team with Mahdi, we found 15 of total issues but due to some strange judging many valid issues were downgraded. And as a result our 1H, 1M and 1L was confirmed valid in the contest.

The contest **validated** issues are
- H-02
- M-02
- L-04

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Potential loss of commission due to commission rate reduction](#H-01)
    - ### [H-02. Unclaimed rewards not reset after exit in `exit_delegation_pool_action`](#H-02)
- ## Medium Risk Findings
    - ### [M-01. Inaccurate reward calculation due to imprecise year duration constant](#M-01)
    - ### [M-02. No way to cancel L1<->L2 messages requests](#M-02)
    - ### [M-03. Potential DoS in stake function](#M-03)
- ## Low Risk Findings
    - ### [L-01. Unstake will revert when contracts are paused](#L-01)
    - ### [L-02. Rewards claim front-running due to delayed L1->L2 minting](#L-02)
    - ### [L-03. Minted tokens will get stuck in starkgate bridge if destination address doesn't fit in felt252](#L-03)
    - ### [L-04. Whitelisted `update_global_index_if_needed` function may delay reward distribution](#L-04)
    - ### [L-05. Missing `l2ToL1MsgHash` function in StarkNet messaging](#L-05)
    - ### [L-06. The timestamp in StarkNet is fully determined by the sequencer.](#L-06)
    - ### [L-07. Missing fee estimation in StarkGate transactions](#L-07)
    - ### [L-08. `addressToUint256Mapping` is missing from `NamedStorage8`](#L-08)
    - ### [L-09. Impossible to deploy the staking contract and the related contracts](#L-09)
    - ### [L-10. Front-running vulnerability in staking reward calculation due to minting curve adjustments](#L-10)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Starknet

### Dates: Sep 16th, 2024 - Oct 4th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-09-starknet-staking)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 3
- Low: 10


# High Risk Findings

## <a id='H-01'></a>H-01. Potential loss of commission due to commission rate reduction            



## Summary

In the current implementation, the commission for staking rewards can **only be decreased**, but this can lead to a loss of commission tokens for the protocol when users claim rewards. Specifically, the protocol does not account for the change in commission rates over time, leading to incorrect commission calculations for periods when the commission rate was higher.

## Vulnerability details

The `calculate_rewards` function in the pool contract calculates rewards using the current commission rate. Here's a simplified view of how the function works:

```Solidity
fn calculate_rewards(
    ref self: ContractState, ref pool_member_info: PoolMemberInfo, updated_index: u64
) -> bool {
    let interest: u64 = updated_index - pool_member_info.index;
    pool_member_info.index = updated_index;
    let rewards_including_commission = compute_rewards_rounded_down(
        amount: pool_member_info.amount, interest: interest
    );
    let commission_amount = compute_commission_amount_rounded_up(
        rewards_including_commission: rewards_including_commission, commission: self.commission.read()
    );
    let rewards = rewards_including_commission - commission_amount;
    pool_member_info.unclaimed_rewards += rewards;
    true
}
```

This function uses the difference between the updated index and the pool member’s previous index to calculate rewards. It then deducts the commission based on the **current** commission rate, which could be lower than the rate at the time the rewards were accumulated.

#### Scenario

* Initial commission: 2%
* User stakes at index 10.
* After 5 blocks (at index 15), the commission is reduced to 1%.
* The user now wants to claim rewards.

The problem occurs because all rewards will be calculated using the new 1% commission rate, even though rewards between index 10 and 15 should be subject to the 2% commission rate. As a result, the protocol will lose part of its expected commission for that period.

## Proof of Concept

1. Alice stakes 100 tokens.
2. Total rewards: 10 tokens.
3. Commission changes from 2% to 1% halfway through.

**Correct Calculation:**

* First half: 5 tokens - 2% = 4.9 tokens
* Second half: 5 tokens - 1% = 4.95 tokens
* Total: 4.9 + 4.95 = 9.85 tokens to Alice, 0.15 tokens as commission.

**Current Pooling Contract Calculation:**

* All rewards: 10 tokens - 1% = 9.9 tokens to Alice, 0.1 tokens as commission.

**Result:**

* Alice receives 0.05 tokens extra.
* The protocol loses 0.05 tokens in commission.

This discrepancy arises because the contract uses only the current commission rate, ignoring historical changes in the commission rate.

## Impact

The protocol can lose part of the commission it is entitled to during periods when the commission rate was higher. This could result in a loss of funds for the protocol, especially if commission changes are frequent.

## Recommendation

I don't have any solid solution but here is what I think you can do;

Record the commission rate at different indices whenever it changes.

Then when calculating rewards, apply the commission rate that was in effect when the rewards were earned, not just the current commission rate.

## <a id='H-02'></a>H-02. Unclaimed rewards not reset after exit in `exit_delegation_pool_action`            



## Github

<https://github.com/Cyfrin/2024-09-starknet-competitive/blob/870c94ad07e8226b6f2383b715f80f82970bf130/workspace/apps/staking/contracts/src/pool/pool.cairo#L211-L246>

<https://github.com/Cyfrin/2024-09-starknet-competitive/blob/870c94ad07e8226b6f2383b715f80f82970bf130/workspace/apps/staking/contracts/src/pool/pool.cairo#L513-L522>

## Summary

A critical issue exists in the `exit_delegation_pool_action` function, where unclaimed rewards are not reset to zero after they are claimed. This oversight can allow malicious users to repeatedly claim rewards, leading to fund drainage from the pool.

## Vulnerability Details

In the `exit_delegation_pool_action` function, the unclaimed rewards of a pool member are sent to their reward address, but the `unclaimed_rewards` field is not reset to zero after the rewards are claimed. This allows a user to repeatedly claim their rewards without a proper reset.

The issue arises from this section of the function:

```rust
// Claim rewards.
self.send_rewards_to_pool_member(
    :pool_member,
    reward_address: pool_member_info.reward_address,
    amount: pool_member_info.unclaimed_rewards,
    :erc20_dispatcher
);
```

After sending rewards, the `unclaimed_rewards` is not set to zero before the pool member's data is either updated or removed.

## Impact

This vulnerability allows pool members to repeatedly claim their full unclaimed rewards by calling the `exit_delegation_pool_action` function multiple times. If the pool member exits with a small amount, the full unclaimed rewards can be claimed repeatedly, draining the pool’s funds.

## Proof of Concept

1. A user joins the delegation pool and accumulates rewards.
2. The user then triggers the `exit_delegation_pool_action` function to claim their rewards, but the `unclaimed_rewards` is not reset.
3. The user can repeatedly call the function with small unstake amounts, claiming the full rewards again and again without resetting the `unclaimed_rewards` value.

This repeated claiming can significantly deplete the pool’s rewards and result in substantial financial loss for the system.

## Recommendation

To resolve the issue, the contract should reset the `unclaimed_rewards` to zero immediately after sending the rewards. The following change is recommended:

```rust
// Claim rewards.
self.send_rewards_to_pool_member(
    :pool_member,
    reward_address: pool_member_info.reward_address,
    amount: pool_member_info.unclaimed_rewards,
    :erc20_dispatcher
);

// Reset unclaimed rewards
pool_member_info.unclaimed_rewards = Zero::zero();

if pool_member_info.amount.is_zero() {
    self.remove_pool_member(:pool_member);
} else {
    pool_member_info.unpool_time = Option::None;
    self.pool_member_info.write(pool_member, Option::Some(pool_member_info));
}
```

This fix ensures that the rewards are properly reset, preventing the exploit from being repeated.

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Inaccurate reward calculation due to imprecise year duration constant            



Github

<https://github.com/Cyfrin/2024-09-starknet-competitive/blob/3769b2364247b07970d6299341f7ff7769123975/workspace/apps/staking/contracts/src/reward_supplier/reward_supplier.cairo#L24>

## Summary

The current implementation of the reward calculation for Starknet staking overestimates daily rewards by not accounting for the extra `0.25` days in a year, resulting in long-term inaccuracies in reward distribution.

## Vulnerability details

1. **Overestimation of Daily Rewards**:
   * The current implementation assumes there are 365 days in a year, whereas a more accurate calculation should consider 365.25 days to account for leap years. This leads to a daily overestimation of rewards by approximately 0.068%.

2. **Cumulative Error**:
   * Over a period of 4 years, this overestimation results in a cumulative error of about 0.274%, which is roughly equivalent to one extra day's worth of rewards being distributed.

3. **Long-term Implications**:
   * This error can lead to the premature depletion of the reward pool and unintended inflation in the token supply, ultimately impacting the tokenomics of the system.

4. **Fairness Concerns**:
   * The reward calculation slightly favors stakers in non-leap years, as they receive more rewards than intended, creating a discrepancy in reward distribution.

## Impact

* **Token Distribution**: Slightly faster than intended, which could lead to incorrect projections in token circulation and affect overall tokenomics.
* **Reward Pool**: Premature depletion of the reward pool due to the over-distribution of rewards.
* **Staker Fairness**: Unequal distribution of rewards, as stakers in non-leap years receive more than they should.

## Recommendation

To correct this issue, it is recommended to use a higher precision constant to represent the length of a year in seconds. The best option is to adjust the `SECONDS_IN_YEAR` constant to use 365.24 days, which will balance accuracy and simplicity while maintaining integer arithmetic.

Cairo doesn't support floating-point literals like 365.251. However, you can achieve a similar result by using integer arithmetic and adjusting your calculation.

```Solidity
pub const SECONDS_IN_YEAR: u128 = (365 * 24 * 60 * 60) + (6 * 60 * 60); // 365 days + 6 hours

fn calculate_rewards(ref self: ContractState) -> u128 {
    let minting_curve_dispatcher = self.minting_curve_dispatcher.read();
    let yearly_mint = minting_curve_dispatcher.yearly_mint();
    let last_timestamp = self.last_timestamp.read();
    let current_time = get_block_timestamp();
    self.last_timestamp.write(current_time);
    let seconds_diff = current_time - last_timestamp;
    yearly_mint * seconds_diff.into() / SECONDS_IN_YEAR
}
```

This adds the extra 6 hours (*0.25 days*) to account for the leap year every four years 2.

## <a id='M-02'></a>M-02. No way to cancel L1<->L2 messages requests            



## Summary

In the current Starknet staking system, there is no mechanism to cancel L1<->L2 messages. If a failure occurs on the L2 side, minted tokens may become stuck in the StarkGate bridge indefinitely, without any ability to recover or cancel the transfer.

***

## Vulnerability details

When calculating staking rewards on the L2 side, the `request_funds_if_needed` function checks if additional tokens need to be minted by evaluating whether available funds are less than the unclaimed rewards plus a buffer (`if credit < debit + threshold`). If more tokens are required, the system requests additional minting by calling `send_mint_request_to_l1_staking_minter`, which sends a mint request to L1.

On the L1 side, the `tick` function in the `RewardSupplier` contract fulfills these L2 requests by minting the required tokens and sending them back to L2 via StarkGate's `depositWithMessage` function. The `msg.value` covers gas fees for the L1-to-L2 message processing.

### Vulnerability:

1. Ethereum contracts can send L1-to-L2 messages via the StarkGate bridge, but there is no [guarantee](https://github.com/crytic/building-secure-contracts/tree/master/not-so-smart-contracts/cairo/l1_to_l2_message_failure#l1-to-l2-message-failure:~:text=In%20Starknet%2C%20Ethereum,cancel%20ongoing%20messages.) the message will be processed by the sequencer on L2. Factors like **sudden gas price increases** can result in insufficient gas, preventing the message from being processed on L2.

2. While StarkGate requires a minimum of 20k wei for L1 message processing, this only ensures that the message is sent, not that the sequencer will process it. If the L2 side fails to process the message, the minted tokens become stuck in the StarkGate bridge indefinitely.

3. Starknet does not provide a mechanism for cancelling such messages or recovering the tokens once they are stuck. Without the ability to cancel or retry the failed message, the tokens remain in limbo.

***

## Impact

If the L2 transaction fails after minting tokens on L1, the tokens remain stuck in the StarkGate bridge with no mechanism to recover them, potentially leading to a permanent loss of tokens.

***

## Proof of Concept

1. Josh stakes tokens on L2, triggering the staking process to request additional tokens from L1 via `calculateStakingRewards`.
2. L1 mints the required tokens and sends them to L2 using the `depositWithMessage` function.
3. The `msg.value` (at least 20k wei) ensures the L1 transfer succeeds, but the L2 side fails to execute the necessary `on_receive` and `l1_handler` functions due to sudden spike in the gas price.
4. The tokens remain stuck in the StarkGate bridge because L2 does not complete the transaction and does not update the internal state (`l1_pending_requested_amount` remains unchanged).
5. Without a cancellation or retry mechanism, the tokens are effectively lost unless manual intervention occurs.

***

## Recommendation

As per the [StarkGate documentation](https://docs.starknet.io/starkgate/automated-actions-with-bridging/#:~:text=on_receive%20must%20return%20true%20for%20the%20deposit%20to%20succeed.%20If%20on_receive%20returns%20false%2C%20or%20if%20the%20recipient%20contract%20does%20not%20include%20the%20on_receive%20function%2C%20the%20depositWithMessage%20function%E2%80%99s%20L1%20handler%20fails.%20The%20user%20can%20recover%20their%20funds%20using%20the%20depositWithMessageCancelRequest%20function.), StarkGate provides the `depositWithMessageCancelRequest` function to recover funds when a message fails to process on the L2 side. It is recommended to implement this function in the Starknet staking protocol to enable token recovery in such cases.

Additionally, ensure that both L1 and L2 cancellation mechanisms are available and robust. Exploring and implementing solutions on both sides will prevent permanent token losses and improve system reliability.

## <a id='M-03'></a>M-03. Potential DoS in stake function            



## Summary

The `stake` [function](https://github.com/Cyfrin/2024-09-starknet-competitive/blob/3769b2364247b07970d6299341f7ff7769123975/workspace/apps/staking/contracts/src/staking/staking.cairo#L131-L212) in the Staking contract checks whether the `operational_address` has been used, reverting if it has. However, there is a potential DoS issue where a staker can exploit the `change_operational_address` function to prevent new stakers from using certain operational addresses by sandwiching their transactions. Additionally, there is no check to prevent inactive stakers from using `change_operational_address`, which could further exacerbate the issue.

***

## Vulnerability Details

The `stake` function ensures that the operational address provided during staking has not already been assigned to another staker. This check prevents the use of the same operational address for multiple stakers:

```cairo
fn stake(
    ref self: ContractState,
    reward_address: ContractAddress,
    operational_address: ContractAddress,
    amount: u128,
    pooling_enabled: bool,
    commission: u16,
) -> bool {
    self.assert_is_unpaused();
    self.roles.only_operator();
    self.update_global_index_if_needed();
    let staker_address = get_tx_info().account_contract_address;
    assert_with_err(self.staker_info.read(staker_address).is_none(), Error::STAKER_EXISTS);
    assert_with_err(
        self.operational_address_to_staker_address.read(operational_address).is_zero(), 
        Error::OPERATIONAL_EXISTS
    );
    // other logic
}
```

If the `operational_address` is already in use (i.e., not zero in the mapping), the transaction reverts with an error: `Error::OPERATIONAL_EXISTS`.

### `change_operational_address` function:

The `change_operational_address` function allows a staker to change their operational address at any time. This function does not check if the staker is in the unstaking process, allowing non-active stakers to change operational addresses as well.

```cairo
fn change_operational_address(
    ref self: ContractState, operational_address: ContractAddress
) -> bool {
    self.assert_is_unpaused();
    self.roles.only_operator();
    self.update_global_index_if_needed();
    let staker_address = get_tx_info().account_contract_address;
    let mut staker_info = self.get_staker_info(:staker_address);
    let old_address = staker_info.operational_address;
    staker_info.operational_address = operational_address;
    self.staker_info.write(staker_address, Option::Some(staker_info));
    self.operational_address_to_staker_address.write(old_address, Zero::zero());
    self.operational_address_to_staker_address.write(operational_address, staker_address);
    // emit event, etc.
}
```

### Potential Exploit:

1. A staker could call `change_operational_address` to assign an operational address.
2. Before another staker stakes with the same operational address, the existing staker could quickly call `change_operational_address` again, effectively “sandwiching” the new staker’s transaction and causing it to revert.
3. This prevents new stakers from using valid operational addresses and disrupts the staking process.

Since there is no check on whether the staker is in the unstaking process, even non-active stakers (those who may have withdrawn or are not staking currently) can use this tactic to lock up valid operational addresses.

**P.S.:** *I understand that with the sequencer currently being centralized, the risk is minimal. However, the StarkNet team has repeatedly mentioned that they will be decentralizing the sequencer very soon, which will introduce a public mempool, and this issue will become relevant in that context.*

***

## Impact

* **Denial of Service:** A malicious or inactive staker can repeatedly block new stakers from using operational addresses, disrupting the staking process.
* **Unfair Advantage:** By strategically sandwiching their transactions, a staker can gain control over certain operational addresses and prevent others from staking.
* **Exploitation by Non-Active Stakers:** Non-active stakers can continue to use `change_operational_address`, which should only be available for active stakers, to further perpetuate the DoS attack.

***

## Recommendations

1. **Restrict `change_operational_address` to Active Stakers:** Implement a check in the `change_operational_address` function to ensure that only active stakers (those who are not in the unstaking process) can call this function.

2. **Introduce a Cooldown or Rate Limiting Mechanism:** To prevent rapid switching of operational addresses, introduce a cooldown period or rate-limiting mechanism for the `change_operational_address` function to reduce the risk of sandwiching attacks.

3. **Check for Pending Transactions:** Before allowing a change in operational address, verify if any staking or unstaking transactions are pending to prevent front-running or sandwich attacks.

By implementing these recommendations, the protocol can mitigate the risk of DoS and improve the overall fairness and reliability of the staking process.


# Low Risk Findings

## <a id='L-01'></a>L-01. Unstake will revert when contracts are paused            



### Summary

The `unstake_intent` function in `staking.cairo` will revert when contracts are paused, preventing users from initiating the unstaking process during this period. This behavior deviates from standard protocol practices where user withdrawals are typically allowed even when contracts are paused.

***

## Vulnerability details

In the StarkNet staking protocol, the `unstake_intent` function includes a check to ensure that the contract is unpaused before allowing users to unstake. This restriction can be seen in the following code snippet:

```cairo
fn unstake_intent(ref self: ContractState) -> u64 {
    self.assert_is_unpaused(); // not allowed to unstake when contracts paused @audit        
    // extra code
}
```

Typically, protocols allow users to withdraw or unstake their tokens even when contracts are paused to ensure users retain control over their assets. However, in this case, the protocol restricts the ability to unstake during the paused state, potentially locking users' funds.

***

## Impact

* Users may be unable to access their staked funds during contract pauses, which could create frustration, especially during emergencies or extended pause periods.
* The inability to unstake during pauses may lead to a loss of user trust in the flexibility of the protocol.

***

## Tools Used

* Code analysis

***

## Recommendations

1. **Allow Unstaking During Pauses:** Modify the `unstake_intent` function to allow users to initiate unstaking even when the contracts are paused. This will align with common practice, ensuring that users can always retrieve their assets.

2. **Pause-Only Critical Functions:** Restrict only critical protocol functions during pauses, allowing users to withdraw their funds without affecting the protocol’s stability.

## <a id='L-02'></a>L-02. Rewards claim front-running due to delayed L1->L2 minting            



## Summary

There is a potential issue in the StarkNet staking system related to reward claim front-running due to the 2-3 [hours](https://book.cairo-lang.org/ch16-04-L1-L2-messaging.html#:~:text=L2%2D%3EL1%20messages,3%2D4%20hours) delay in L2->L1 communication for minting new tokens. This delay can lead to unfair reward distribution and exploitation opportunities for faster users who can deplete the available reward pool, leaving others to wait for new tokens.

## Vulnerability details

In the current setup, when a user claims rewards on L2, the rewards are calculated based on the available unclaimed tokens. However, if the available tokens are insufficient, a [request](https://github.com/Cyfrin/2024-09-starknet-competitive/blob/3769b2364247b07970d6299341f7ff7769123975/workspace/apps/staking/contracts/src/reward_supplier/reward_supplier.cairo#L93) is made to mint more tokens from L1. According to starknet this L2->L1 interaction takes around 2-3 hours to finalize, causing a delay in minting new rewards.

### Exploit Scenario:

1. Two users (*User A and User B*) have earned equal rewards.
2. The available unclaimed rewards are insufficient for both users.
3. User A claims rewards first, depleting the available pool.
4. User B must wait 2-3 hours for the L1 minting process to complete, resulting in a delay in receiving their rewards.

This creates an opportunity for front-running by more active or faster users, giving them an unfair advantage. Additionally, users who experience delays may face the risk of price volatility (*e.g., STRK price changes during the waiting period*), impacting the value of their rewards.

**P.S.:** *While the sequencer is currently centralized, the StarkNet team has frequently stated that decentralization is on the horizon. Once the sequencer becomes decentralized and a public mempool is introduced, front-running could become a significant concern*.

## Impact

* **Delayed Rewards:** Slower users face delays in receiving their rewards, leading to frustration and a poor user experience.
* **Unfair Advantage:** Faster users can front-run reward claims and deplete the available pool, creating an imbalance in reward distribution.
* **Price Volatility Risk:** Users who claim rewards later may be exposed to STRK price fluctuations, affecting the value of their rewards.

## Recommendation

Work on reducing the L2->L1 communication delay to ensure more timely updates to the total supply and reward calculations.

Introduce a fair queuing mechanism for reward claims to prevent faster users from depleting the available reward pool before others can claim their share.

Implement a system where a buffer of unclaimed rewards is maintained on L2, ensuring that rewards can be claimed immediately without waiting for L1 minting.

## <a id='L-03'></a>L-03. Minted tokens will get stuck in starkgate bridge if destination address doesn't fit in felt252            



## Summary

The `mintDestination` address, which is set by the admin via the [`setMintDestination`](https://github.com/Cyfrin/2024-09-starknet-competitive/blob/3769b2364247b07970d6299341f7ff7769123975/workspace/apps/staking/solidity/RewardSupplierStorage.sol#L73) function, lacks sanity checks to ensure it is within the size limit of `felt252`. If this address is set to a value larger than `felt252`, it can cause a Denial of Service when interacting with StarkNet L2, as certain functions will fail when they attempt to process addresses larger than `felt252`. It do `require(mintDestination_ != 0, "INVALID_MESSAGE_DISPATCHER");` check but the sanity check for address is missing.

## Vulnerability Details

The `mintDestination` address is used when processing L2 requests and minting tokens to send them back to L1 via StarkGate. The issue arises because the `setMintDestination` function allows the admin to set a `mintDestination` address of type `uint256`. However, when interacting with StarkNet L2, addresses must be of type `felt252`. If the `mintDestination` is set to a value larger than `felt252`, the contract will fail to execute certain functions, particularly the `#[l1_handler] fn update_total_supply`, which will revert, causing the minted tokens to become stuck in the StarkGate bridge.

### Code Analysis

```Solidity
function setMintDestination(uint256 mintDestination_) internal {
    NamedStorage.setUintValueOnce(L2_MINT_DESTINATION_TAG, mintDestination_);
}
```

The `mintDestination_` is set as a `uint256`, which may exceed the maximum size of `felt252` used on the StarkNet side.

The following function uses the `from_address` parameter, which is of type `felt252`, to verify the L1-to-L2 transaction:

```Solidity
#[l1_handler]
fn update_total_supply(ref self: ContractState, from_address: felt252, total_supply: u128) {
        assert_with_err(
            from_address == self.l1_staking_minter_address.read(),
            Error::UNAUTHORIZED_MESSAGE_SENDER
        );
        let old_total_supply = self.total_supply.read();
        self.total_supply.write(total_supply);
        self.emit(Events::TotalSupplyChanged { old_total_supply, new_total_supply: total_supply });
}
```

If the `mintDestination` set on L1 is larger than the `felt252` type, the `update_total_supply` function will revert, as the address will not fit within the constraints of `felt252`.

### Scenario

1. The admin sets the `mintDestination` to an address larger than the size of `felt252`.
2. When a request to mint tokens is processed and sent back to L1, StarkGate sequencer calls the `update_total_supply` function.
3. Since the `mintDestination` is larger than `felt252`, the function will revert, and the tokens will become stuck in the StarkGate bridge, causing a DoS for the transaction.

### Impact

If the `mintDestination` is set to an address larger than `felt252`, the protocol will experience a DoS, as the StarkNet L2 functions that rely on address verification, such as `update_total_supply`, will revert. This will prevent minted tokens from being transferred back to L1, leaving them stuck in the StarkGate bridge and halting further processing.

**PS***: While the likelihood of this issue is low because the `mintDestination` is set by an admin, it’s important to note that it has not been mentioned in any docs or **contest README** that the admin is aware of this size limitation. Since a valid ETH address can fit into `uint256` but may exceed the `felt252` limit, an admin might unintentionally input an invalid address. If the address exceeds the size of `felt252`, the impact of this issue becomes critical, as it leads to the permanent locking of minted STRK tokens in the StarkGate bridge. ***Additionally, there is no current mechanism to cancel or revert this message due to the lack of a messaging feature in StarkNet staking***.*

### Recommendation

Add a sanity check in the `setMintDestination` function to ensure that the `mintDestination` address fits within the size limit of `felt252`. This would prevent any addresses larger than the allowed size from being set, thus avoiding the potential DoS scenario.

## <a id='L-04'></a>L-04. Whitelisted `update_global_index_if_needed` function may delay reward distribution            



## Summary

The `update_global_index_if_needed` function in `operator.cairo`, which is responsible for maintaining accurate reward calculations, can only be called by whitelisted addresses. This restriction poses a risk of delayed reward updates if the whitelisted entities fail to trigger it regularly. This could result in unequal reward distribution and negatively affect the protocol's decentralization goals.

**PS:** *Do note that `update_global_index_if_needed`function is in operator.cairo but it is directly impacting the staking contract*

## Vulnerability details

The `update_global_index_if_needed` function is crucial for ensuring that rewards are calculated correctly and distributed based on up-to-date information.

```Solidity
fn update_global_index_if_needed(ref self: ContractState) -> bool {
      self.check_whitelist(get_caller_address()); // ❌ this check is not needed here @audit 
      self.staking_dispatcher.read().update_global_index_if_needed()
}
```

However, it is currently limited to being callable only by whitelisted addresses. This reliance on a small set of entities creates a potential bottleneck if those whitelisted callers do not update the index frequently enough. Furthermore, this restriction is contrary to decentralization principles, as it prevents regular users or pool members from contributing to the system's maintenance.

### Key points:

* **Functionality:** Only whitelisted addresses can call `update_global_index_if_needed`.
* **Risk:** If whitelisted entities do not call the function frequently, it could lead to larger-than-expected reward accumulations or delays in reward distribution.
* **Centralization:** Restricting the function to certain addresses increases dependency on specific actors, reducing the decentralization of the protocol.

## Impact

* **Delayed Reward Distribution:** If the index isn't updated frequently, users may experience delays in receiving accurate reward calculations, leading to unequal or inconsistent reward payouts.
* **Potential Exploits:** Larger, unexpected reward claims may occur, creating an imbalance in the reward distribution system.
* **Increased Centralization:** The restriction to whitelisted addresses creates a centralized point of control for a critical system function, reducing the network's overall decentralization.

## Recommendation

Allow any user to call `update_global_index_if_needed` to ensure the function is triggered regularly, even in low-activity periods. 

## <a id='L-05'></a>L-05. Missing `l2ToL1MsgHash` function in StarkNet messaging            



## Summary

The reward minting process of the cross-layer communication between ETH (L1) and StarkNet (L2) is broken. The `tick` function, which is responsible for processing token mint requests from L2, relies on a non-existent `l2ToL1MsgHash` [function](https://github.com/Cyfrin/2024-09-starknet-competitive/blob/3769b2364247b07970d6299341f7ff7769123975/workspace/apps/staking/solidity/RewardSupplier.sol#L110) in the StarkNet messaging contract. This discrepancy will cause the `tick` function to revert, effectively breaking the token minting mechanism.

## Vulnerability Details

The `tick` function in the reward supplier contract attempts to use `messagingContract().l2ToL1MsgHash` to compute and verify the message hash. However, this function does not exist in the [StarkNet core messaging contract](https://github.com/starkware-libs/cairo-lang/blob/4e233516f52477ad158bc81a86ec2760471c1b65/src/starkware/starknet/eth/StarknetMessaging.sol#L130C7-L130C7#). The relevant code snippet is as follows:

```solidity
bytes32 msgHash = messagingContract().l2ToL1MsgHash(
    source(),
    address(this),
    messageReceived
);
```

The StarkNet core messaging contract (StarknetMessaging.sol) does not provide an `l2ToL1MsgHash` function. Instead, it uses internal hash computation within functions like `consumeMessageFromL2`.

## Impact

The impact of this vulnerability is severe:

1. **Broken Functionality**: The `tick` function will consistently revert due to the missing function, rendering the token minting process inoperable.

2. **Stuck Requests**: Mint requests originating from L2 will never be processed on L1, leading to a backlog of unfulfilled requests.

3. **User Experience**: Users expecting to receive minted tokens will face indefinite delays, potentially leading to loss of trust in the system.

4. **Economic Implications**: The inability to mint tokens as requested could disrupt the token economy and any dependent systems or protocols.

5. **System Inconsistency**: This issue creates a state inconsistency between L1 and L2, where L2 expects actions that L1 cannot perform.

## Root Cause

The root cause appears to be a mismatch between the expected interface of the messaging contract and the actual implementation provided by StarkNet's core messaging contract. This suggests either:

1. A misunderstanding of the StarkNet messaging protocol.
2. Use of an outdated or incorrect interface for the messaging contract.
3. Reliance on a custom function that was not implemented in the final messaging contract.

## Recommendations

Utilize the existing functions in the StarkNet core messaging contract to calculate and verify message hashes. For example, adapt the logic used in `consumeMessageFromL2` for hash computation.

If a custom `l2ToL1MsgHash` function is necessary, create a new contract that extends the core StarkNet messaging functionality. This contract should:

* Inherit from or wrap the core StarkNet messaging contract.
* Implement the required `l2ToL1MsgHash` function.
* Be deployed separately and set as the `messagingContract` address.


## <a id='L-06'></a>L-06. The timestamp in StarkNet is fully determined by the sequencer.            



## Summary

The `block.timestamp` value is widely used in the codebase to validate critical time-dependent operations. However, in Starknet, there are no current restrictions on the return values of the `get_block_timestamp()` function. This allows the sequencer to submit an arbitrary or incorrect timestamp, which can lead to vulnerabilities in key operations such as staking and unstaking.

## Vulnerability Details

In the staking module, the function `get_block_timestamp()` is used to manage the time-based logic for unstaking, which can create a security risk. When a user calls `unstake_intent`, it records the intention to unstake and sets the unstake time to `21 days + current time`. This means the user cannot unstake until after the 21-day lockup period has passed.

The issue arises when the user calls [`unstake_action`](https://github.com/Cyfrin/2024-09-starknet-competitive/blob/3769b2364247b07970d6299341f7ff7769123975/workspace/apps/staking/contracts/src/staking/staking.cairo#L329-L331) to complete the unstaking process. The `unstake_action` function checks whether the 21-day period has passed using the following assertion:

```Solidity
assert_with_err(
    get_block_timestamp() >= unstake_time, 
    Error::INTENT_WINDOW_NOT_FINISHED
);
```

Since the sequencer controls the block timestamp and there are no enforced restrictions, it could potentially return an incorrect timestamp, bypassing this check.

### Exploit Scenarios:

1. **Bypassing the 21-Day Lockup Period:**\
   A malicious sequencer could return a timestamp that is greater than or equal to the `unstake_time` even if the 21-day period has not passed, allowing the user to unstake prematurely.

2. **Preventing Unstaking After 21 Days:**\
   Conversely, the sequencer could return a timestamp that is less than the actual time, even after 21 days, preventing the user from unstaking even though the lockup period has elapsed.

## Impact

The impact of this vulnerability could lead to two potential scenarios:

* **Premature Unstaking:** A user might be able to unstake before the intended 21-day lockup period, violating the protocol’s intended behavior.
* **Delayed Unstaking:** A user could be unfairly restricted from unstaking after 21 days due to an incorrect timestamp, leading to user frustration and a loss of trust in the protocol.

Both scenarios can significantly undermine the protocol's integrity and affect user experience.

## Recommendation

### Short-Term Solution:

Minimize reliance on the `block.timestamp` value for critical, time-sensitive operations. Wherever possible, use alternative methods or implement additional validation checks that do not depend solely on the timestamp.

### Long-Term Solution:

Keep up to date with the latest Starknet documentation to monitor any changes or improvements to the handling of `block.timestamp`. As Starknet matures, any improvements to timestamp handling by the sequencer should be integrated into the codebase to ensure security and reliability.

## <a id='L-07'></a>L-07. Missing fee estimation in StarkGate transactions            



### Summary

StarkGate enforces a [minimum fee](https://docs.starknet.io/starkgate/estimating-fees/) for all transactions to cover the costs of L1 → L2 messages. Fees can be estimated using the `estimateDepositFeeWei` and `estimateEnrollmentFeeWei` functions. However, when the `tick` function is called by the user, it does not perform fee estimation, which can lead to transaction failures if the provided fee is less than the required minimum.

***

## Vulnerability Details

In StarkGate, transactions from L1 to L2 require a minimum fee to ensure the message processing cost is covered. This minimum fee can be estimated using the provided `estimateDepositFeeWei` and `estimateEnrollmentFeeWei` functions. However, the `tick` function, which is responsible for minting and sending rewards from L1 to L2, does not call these fee estimation functions to ensure that the correct fee is provided.

As a result, if the user provides a fee that is below the required minimum, the transaction will fail. This issue becomes more significant if there is no clear guidance on the required fee, leading to repeated failures due to insufficient fees.

***

## Impact

* **Transaction Failure:** Every time the `tick` function is called without proper fee estimation, the transaction will fail if the provided fee is below the required minimum.
* **Poor User Experience:** Users may experience frustration due to repeated transaction failures without understanding the cause, resulting in delays in minting rewards.
* **Inefficient Use of Resources:** Repeated failed transactions waste resources, including gas fees, without achieving the intended outcome.

***

## Recommendation

1. **Implement Fee Estimation in `tick`:** Update the `tick` function to call `estimateDepositFeeWei` or `estimateEnrollmentFeeWei` before proceeding with the transaction. This will ensure that users provide an adequate fee to cover the L1 → L2 message costs.

2. **Provide Fee Guidance:** Include clear documentation or messaging to guide users on the appropriate fee required for calling the `tick` function, reducing the likelihood of transaction failures.

## <a id='L-08'></a>L-08. `addressToUint256Mapping` is missing from `NamedStorage8`            



## Summary

The `MintManager` contract relies on a `addressToUint256Mapping` [function](https://github.com/Cyfrin/2024-09-starknet-competitive/blob/3769b2364247b07970d6299341f7ff7769123975/workspace/apps/staking/solidity/MintManagerStorage.sol#L19) in the `NamedStorage8` [library](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/solidity/libraries/NamedStorage8.sol), which is not implemented. This function is crucial for managing allowances in the contract.

## Impact

* Critical: The contract will fail to compile due to the missing function.
* If deployed using an alternative compilation method:
  * Severe disruption of core contract functionality
  * Potential data corruption and inconsistent contract state
  * Possible security vulnerabilities due to incorrect storage access
  * All allowance-related operations (minting, approving, increasing, decreasing) will malfunction

## Recommendation

1. Implement the missing `addressToUint256Mapping` function in the `NamedStorage8` library:

   ```solidity
   function addressToUint256Mapping(string memory tag_)
       internal
       pure
       returns (mapping(address => uint256) storage randomVariable)
   {
       bytes32 location = keccak256(abi.encodePacked(tag_));
       assembly {
           randomVariable.slot := location
       }
   }
   ```

2. Alternatively, modify the `allowance()` function in `MintManagerStorage` to use an existing mapping function, such as:

   ```solidity
   function allowance() internal pure returns (mapping(address => uint256) storage) {
       return NamedStorage.bytes32ToUint256Mapping(ALLOWANCE_TAG);
   }
   ```

   This would require converting addresses to bytes32 when accessing the mapping.

3. After implementing the fix, thoroughly test all allowance-related functionalities to ensure correct operation.

## <a id='L-09'></a>L-09. Impossible to deploy the staking contract and the related contracts            



## Summary

There is currently an interdependency issue between the [staking contract](https://github.com/Cyfrin/2024-09-starknet-competitive/blob/3769b2364247b07970d6299341f7ff7769123975/workspace/apps/staking/contracts/src/staking/staking.cairo#L100-L127), the minting\_curve [contract](https://github.com/Cyfrin/2024-09-starknet-competitive/blob/3769b2364247b07970d6299341f7ff7769123975/workspace/apps/staking/contracts/src/minting_curve/minting_curve.cairo#L53-L67), and the reward\_supplier [contract](https://github.com/Cyfrin/2024-09-starknet-competitive/blob/3769b2364247b07970d6299341f7ff7769123975/workspace/apps/staking/contracts/src/reward_supplier/reward_supplier.cairo#L56-L80), making it impossible to deploy the staking contract.

## Vulnerability details

The interdependency occurs due to the following:

1. **Staking Contract**: The staking contract requires the `reward_supplier` address during its construction.
2. **Reward Supplier Contract**: The reward\_supplier contract, in turn, requires the `staking` and `minting_curve` contract addresses for its initialization.
3. **Minting Curve Contract**: The minting\_curve contract also requires the `staking` contract address during its construction.

This interdependency leads to a situation where the contracts cannot be deployed in the correct order without causing issues.

## Impact

Due to the interdependencies, it is impossible to deploy the staking contract and the related contracts, effectively halting the deployment of the entire staking system.

## Recommendations

A potential solution is to modify the **minting\_curve** and **reward\_supplier** contracts so that they can set the staking contract address via setters functions after deployment, rather than requiring it in their constructors.

Additionally, the contracts should implement a safeguard mechanism to ensure the `staking` contract address has been properly set before any functions that rely on it can be used.

This approach breaks the interdependency and ensures that all contracts can be deployed in the correct order, while also maintaining the integrity of the system.

## <a id='L-10'></a>L-10. Front-running vulnerability in staking reward calculation due to minting curve adjustments            



## Summary

The [minting curve](https://community.starknet.io/t/snip-18-staking-s-first-stage-on-starknet/114334#p-2357184-minting-curve-7) in the Starknet staking system adjusts rewards based on the number of stakers, decreasing rewards as more people stake and increasing them when fewer tokens are staked. A staker can potentially exploit this system by front-running large staking transactions, claiming rewards before the reward rate drops due to an influx of new stakers.

## Impact

This creates an unfair advantage for stakers who can predict incoming large transactions, allowing them to claim higher rewards than they would otherwise receive after the reward adjustment. This could lead to imbalances in the reward distribution and undermine fairness in the staking system.

## Recommendation

Introduce a mechanism to prevent or mitigate front-running, such as randomizing reward calculation intervals or setting a lock period for reward claims after significant staking events, ensuring fairer distribution of rewards among all stakers.


