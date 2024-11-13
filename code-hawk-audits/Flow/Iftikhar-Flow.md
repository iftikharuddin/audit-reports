# Flow - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Stream balance front-running vulnerability in NFT transfer](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Protocol fee front-running undermines business strategy](#M-01)
    - ### [M-02. Stream can be voided in pause state also](#M-02)
    - ### [M-03. Invariant check is not implemented in the code](#M-03)
- ## Low Risk Findings
    - ### [L-01. Missing EIP-4906 interface support in Flow NFT Implementation](#L-01)
    - ### [L-02. Stream cannot be restarted with rps 0 due to rate change restriction](#L-02)
    - ### [L-03. No way to refund max](#L-03)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Sablier

### Dates: Oct 25th, 2024 - Nov 1st, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-10-sablier)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 3
- Low: 3


# High Risk Findings

## <a id='H-01'></a>H-01. Stream balance front-running vulnerability in NFT transfer            



## Summary

Stream NFT holders are vulnerable to front-running attacks where stream senders can withdraw all funds just before an NFT purchase transaction, leaving buyers with empty streams.

## Vulnerability details

The `_withdraw` function allows stream senders to withdraw any available balance without restrictions, even when the stream NFT is listed for sale. There's no mechanism to lock funds or prevent withdrawals during NFT transfers.

Key code:

```solidity
function _withdraw(uint256 streamId, address to, uint128 amount) internal {
    // Only checks recipient ownership
    if (to != _ownerOf(streamId) && !_isCallerStreamRecipientOrApproved(streamId)) {
        revert Errors.SablierFlow_WithdrawalAddressNotRecipient();
    }
    // No checks for NFT transfer status
    _streams[streamId].balance -= amount;
    token.safeTransfer({ to: to, value: amount });
}
```

Streams can be created with a `transferable` option, allowing the sender to list them on NFT marketplaces. However, this setup introduces a risk: before a buyer completes the purchase, the stream’s owner could front-run the transaction by withdrawing all deposited funds, leaving the buyer with an empty stream\\

## Impact

HIGH. Buyers can lose significant funds by purchasing seemingly funded stream NFTs that get drained before transfer completes.

## Likelihood

HIGH. The attack requires minimal setup and standard MEV tools can easily monitor and front-run NFT purchase transactions.

## Proof of concept

Let's do a pseudo code PoC

```Solidity
function exploit() external {
    // Alice creates stream with 2000 USDC
    uint256 streamId = flow.create(alice, alice, ratePerSec, USDC, true);
    flow.deposit(streamId, 2000e6);
    
    // Alice lists NFT on marketplace
    marketplace.list(streamId, price);
    
    // Bob attempts to buy it seeing that it has 2000 USDC
    // Alice front-runs with:
    flow.withdrawMax(streamId, alice);
    
    // Bob's purchase completes but stream is empty
    marketplace.buy(streamId) // Bob receives worthless NFT
}
```

## Recommendation

1. Add a "locked" state when NFT is listed:

```solidity
struct Stream {
    bool isListed;
    uint256 lockedBalance;
}

function _withdraw() internal {
    require(!stream.isListed, "Funds locked during listing");
    // ... rest of withdraw logic
}
```

1. Implement atomic NFT+funds transfers where marketplace handles both NFT and stream balance transfer in single transaction.

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Protocol fee front-running undermines business strategy            



## Github

<https://github.com/Cyfrin/2024-10-sablier/blob/8a2eac7a916080f2022527408b004578b21c51d0/src/abstracts/SablierFlowBase.sol#L262>

## Summary

Although the protocol intentionally applies new fees retroactively to all pending withdrawals as a business strategy, this mechanism can be circumvented through front-running by attackers. Users monitoring the mempool can detect fee increase transactions and quickly withdraw their funds before the new fee takes effect.

## Impact

MEDIUM - While no funds are at risk, this edge case:

* Reduces protocol revenue
* Creates unfair advantage for tech-savvy users
* Makes fee changes unpredictable
* Undermines intended business model, so basically protocol loses the expected fees. 

## What causes it?

```solidity
function _withdraw(streamId, to, amount) internal {
    // ... other checks ...
    
    // Fee is checked at withdrawal time
    UD60x18 protocolFee = protocolFee[token];
    
    if (protocolFee > ZERO) {
        // Current fee rate applied to full amount
        (protocolFeeAmount, amount) = calculateAmountsFromFee(amount, fee);
        protocolRevenue[token] += protocolFeeAmount;
    }
}
```

The fee is determined at withdrawal time rather than when tokens were streamed, making it vulnerable to front-running when fees are increased.

## Likelihood

LOW-MED - This attack is:

* Easy to execute
* Highly profitable
* Requires only mempool monitoring
* Can be automated
* Most likely during significant fee increases which will be not frequent ( so low likelihood )

## Proof of Concept

front-running Scenario ( where user avoids high fees ):

1. Admin: Let say initial fee is set to 0 % and then admin changed it to setProtocolFee(token, 10%), now transaction is in mempool
2. User: Spots this transaction in mempool
3. User: Quickly submits withdrawal with higher gas
   * withdraw(streamId, fullAmount) // Front-runs with higher gas
4. Admin's fee change executes after
5. User gets funds with original lower fee (0%)

## Recommendation

There can be many solutions for mitigating these kinda frontrunning issues, the one I have in mind is to use Flashbots or any private memepool service for updating fees like this.

## <a id='M-02'></a>M-02. Stream can be voided in pause state also            



## Summary

The `void` function lacks a `notPaused` modifier, allowing streams to be permanently voided during temporary maintenance pauses. Despite protocol's assumption of trust between all entities (sender, recipient, and approved operators), this can lead to unintentional permanent termination of streams that were only meant to be temporarily paused.

## Vulnerability Details

```Solidity
function void(uint256 streamId)
    external
    override
    noDelegateCall
    notNull(streamId)
    notVoided(streamId)  // Only checks if not already voided - @audit
    updateMetadata(streamId)
{
    _void(streamId);
}

// Missing crucial check in _void():
function _void(uint256 streamId) internal {
    // No check if stream is paused
    if (msg.sender != _streams[streamId].sender && 
        !_isCallerStreamRecipientOrApproved(streamId)) {
        revert Errors.SablierFlow_Unauthorized(...);
    }
    _streams[streamId].isVoided = true;
    _streams[streamId].ratePerSecond = ud21x18(0);
    // Stream can never be restarted
}
```

**Important Note**: While the readme docs states:

> It is assumed that a trust relationship is formed between the sender, recipient, and approved operators participating in a stream.

This issue can occur even with complete trust between parties, as it stems from normal operational activities rather than malicious intent.

## Impact

HIGH. Because:

1. Paused streams can be permanently voided by trusted parties acting in good faith
2. Once voided, streams cannot be restarted
3. Affects payroll and payment systems during routine maintenance
4. Forces creation of new streams after maintenance
5. Disrupts accounting and stream history

## Likelihood

LOW. This can occur in in business operations when:

1. System maintenance requiring temporary pauses
2. Different departments responding to stream states
3. Communication gaps in standard procedures

## Proof of Concept

Let's do a pseudo code PoC as example

```solidity
// Scenario: Trusted parties, routine maintenance
function testTrustedPartiesMaintenanceScenario() public {
    // Setup: All parties trust each other
    uint256 streamId = flow.create(sender, recipient, rate, token, true);
    
    // Finance department pauses for maintenance
    vm.prank(sender);  // Finance dept
    flow.pause(streamId);
    assertEq(flow.statusOf(streamId), PAUSED_SOLVENT);
    
    // HR department (trusted recipient) sees non-working stream
    vm.prank(recipient);  // HR dept
    flow.void(streamId);  // Acting in good faith
    
    // Finance can't restart after maintenance
    vm.expectRevert("Stream voided");
    flow.restart(streamId, rate);
}
```

## Recommendation

Add `notPaused` modifier to `void`:

```solidity
function void(uint256 streamId)
    external
    override
    noDelegateCall
    notNull(streamId)
    notVoided(streamId)
    notPaused(streamId)  // Add this modifier
    updateMetadata(streamId)
{
    _void(streamId);
}
```


## <a id='M-03'></a>M-03. Invariant check is not implemented in the code            



## Summary

The protocol has an invariant that requires `token.balanceOf(SablierFlow) >= aggregateBalance[token]`, but lacks explicit validation in the code. While this may work with standard ERC20s, it could break with non-standard tokens.

Please do note that I am aware of the tokens like FoT, and rebase tokens are OOS but the invariant can be breaked if these tokens are used because it is not handled in the code.

## Vulnerability Details

The invariant check is missing from `withdraw`, `deposit`, and \`refund\`

Consider this real-world analogy:

```Solidity
Bank System:
- Ledger Balance (aggregateBalance): What bank thinks it has
- Actual Cash (token.balanceOf): What's actually in vault
- Requirement: Actual Cash >= Ledger Balance
```

Current Implementation Problem:

```solidity
function _deposit(uint256 streamId, uint128 amount) internal {
    // PROBLEM: Updates ledger before checking vault
    aggregateBalance[token] += amount;        // Ledger: +100
    token.safeTransferFrom(..., amount);      // Vault: might get less than 100
}
```

Real World Example:

```solidity
// Initial State
Ledger (aggregateBalance): 1000 USDT
Vault (token.balanceOf): 1000 USDT

// Deposit 100 USDT with 1% fee token
1. Updates Ledger: 1000 + 100 = 1100 USDT
2. Actual Transfer: 99 USDT received (1% fee)
3. Final State:
   Ledger: 1100 USDT
   Vault: 1099 USDT
   // Bank thinks it has more than it does!
```

## Impact

HIGH - Like a protocol promising more money than it has:

1. Token balance inconsistencies can occur
2. Deflationary/fee-on-transfer tokens could break accounting
3. Protocol could promise more tokens than it has
4. Multiple streams could become insolvent

&#x20;

## Proof of concept

Let's do some pseudo code PoC

```Solidity
function testInvariantBreak() public {
    // Setup fee token (1% fee)
    FeeToken token = new FeeToken();
    
    // Create stream
    uint256 streamId = flow.create(sender, recipient, rate, token, true);
    
    // Initial deposit 1000
    flow.deposit(streamId, 1000);
    // Contract gets 990, but aggregateBalance = 1000
    
    // Check invariant
    assertLt(
        token.balanceOf(address(flow)),  // 990
        flow.aggregateBalance(token)      // 1000
    );
}
```

## Recommendation

Add a generic invariant validation function like below and call it in all functions where it is needed e.g in withdraw, refund and deposit.

```Solidity
function _validateBalanceInvariant(IERC20 token) internal view {
    uint256 actualBalance = token.balanceOf(address(this));
    uint256 expectedMinBalance = aggregateBalance[token];
    require(
        actualBalance >= expectedMinBalance,
        "Balance invariant violation"
    );
}
```

This ensures:

&#x20;

1. Actual token balance ≥ tracked balance
2. Accounting remains correct with non-standard tokens
3. Early detection of balance inconsistencies
4. Protocol solvency guarantee


# Low Risk Findings

## <a id='L-01'></a>L-01. Missing EIP-4906 interface support in Flow NFT Implementation            



## Summary

The SablierFlowBase contract emits EIP-4906 metadata update events but fails to properly implement the [EIP-4906](https://eips.ethereum.org/EIPS/eip-4906#reference-implementation:~:text=The%20supportsInterface%20method%20MUST%20return%20true%20when%20called%20with%200x49064906.) interface, specifically lacking the required `supportsInterface(bytes4)` function that should return true for interface ID `0x49064906`. This prevents NFT marketplaces and other protocols from detecting the contract's metadata update capabilities.

## Vulnerability Details

The SablierFlowBase contract inherits from ERC721 and uses EIP-4906 events for metadata updates:

```solidity
abstract contract SablierFlowBase is
    Adminable,
    ISablierFlowBase,
    ERC721
{
    // Uses EIP-4906 events
    modifier updateMetadata(uint256 streamId) {
        _;
        emit MetadataUpdate({ _tokenId: streamId });
    }

    function setNFTDescriptor(...) {
        // ...
        emit BatchMetadataUpdate({ _fromTokenId: 1, _toTokenId: nextStreamId - 1 });
    }
}
```

However, the contract doesn't implement the required `supportsInterface(bytes4)` function to signal EIP-4906 support. This function should return true for the EIP-4906 interface ID (`0x49064906`).

## Impact

HIGH. Stream NFTs contain critical information that must stay updated:

1. Stream status (STREAMING\_SOLVENT, STREAMING\_INSOLVENT, etc.)
2. Current balance and withdrawal amounts
3. Rate per second and duration details
4. Covered/uncovered debt amounts

Without proper EIP-4906 interface support:

* NFT marketplaces won't detect metadata update capabilities
* Stream NFT displays may show stale data after state changes
* Buyers might make decisions based on outdated stream information
* Critical stream parameters (balance, status) might not refresh properly

This is especially problematic for transferable streams where accurate metadata is crucial for secondary market trading.

## Likelihood

MEDIUM. This affects all transferable stream NFTs in the protocol, and the issue will manifest whenever:

* Stream parameters are updated
* Deposits or withdrawals occur
* Stream status changes
* NFTs are listed on marketplaces

## Proof of Concept

Below is pseudo code PoC

```Solidity
// Test to demonstrate missing interface support
function testEIP4906Support() public {
    // Deploy SablierFlow
    SablierFlow flow = new SablierFlow(admin, descriptor);
    
    // Check EIP-4906 interface support
    bool supportsEIP4906 = flow.supportsInterface(0x49064906);
    assertEq(supportsEIP4906, false); // Fails - should return true - @audit
    
    // Create and modify stream
    uint256 streamId = flow.create(...);
    flow.deposit(streamId, 100);
    
    // Marketplace integration would fail to detect metadata updates
}
```

## Recommendation

Implement EIP-4906 interface support in SablierFlowBase as the standard is doing [here](https://eips.ethereum.org/EIPS/eip-4906#reference-implementation:~:text=issues%20were%20found-,Reference%20Implementation,-//%20SPDX%2DLicense%2DIdentifier)

```solidity
abstract contract SablierFlowBase is Adminable, ISablierFlowBase, ERC721 {
    function supportsInterface(bytes4 interfaceId)
        public
        view
        virtual
        override
        returns (bool)
    {
        return
            interfaceId == 0x49064906 || // EIP-4906
            super.supportsInterface(interfaceId);
    }
}
```

This ensures:

1. Proper interface detection by NFT marketplaces
2. Accurate metadata updates for stream NFTs
3. Compliance with EIP-4906 specification
4. Better integration with third-party protocols

The fix is simple to implement and critical for the proper functioning of transferable stream NFTs in secondary markets.

## <a id='L-02'></a>L-02. Stream cannot be restarted with rps 0 due to rate change restriction            



&#x20;

## Summary

When a stream is paused (RPS = 0), attempting to restart it with RPS = 0 will revert due to the `_adjustRatePerSecond` function requiring the new rate to be different from the current rate. This prevents legitimate use cases where a sender might want to initialize a funded but non-streaming state after a pause.

## Vulnerability Details

The issue occurs in the following flow:

1. Stream is paused (*sets RPS to 0*):

```solidity
function _pause(uint256 streamId) internal {
    _adjustRatePerSecond({ streamId: streamId, newRatePerSecond: ud21x18(0) });
}
```

1. When trying to restart with RPS = 0:

```Solidity
function _restart(uint256 streamId, UD21x18 ratePerSecond) internal {
    // Checks stream is paused (RPS == 0)
    if (_streams[streamId].ratePerSecond.unwrap() != 0) {
        revert Errors.SablierFlow_StreamNotPaused(streamId);
    }

    // Will revert if new RPS = current RPS (0) - @audit
    _adjustRatePerSecond({ streamId: streamId, newRatePerSecond: ratePerSecond });
}

function _adjustRatePerSecond(uint256 streamId, UD21x18 newRatePerSecond) internal {
    if (newRatePerSecond.unwrap() == _streams[streamId].ratePerSecond.unwrap()) {
        revert Errors.SablierFlow_RatePerSecondNotDifferent(streamId, newRatePerSecond);
    }
}
```

## Impact

* Prevents legitimate use case of restarting a stream in a funded but non-streaming state
* Forces users to set non-zero RPS temporarily just to restart
* Additional gas costs from unnecessary RPS changes
* Poor UX for users wanting to manage stream funds before activating streaming

## Likelihood

LOW - This would impact any user trying to:

1. Pause a stream
2. Fund it without immediate streaming
3. Restart with RPS = 0 to maintain funded but inactive state

## Proof of concept

Let's PoC it with below pseudo code:

```Solidity
function testCannotRestartWithZeroRPS() public {
    // Create stream with RPS = 1
    uint256 streamId = createStream(sender, recipient, ud21x18(1e18));
    
    // Pause stream (sets RPS to 0)
    flow.pause(streamId);
    
    // Try to restart with RPS = 0, this will revert bcz of similar rps
    vm.expectRevert(Errors.SablierFlow_RatePerSecondNotDifferent.selector);
    flow.restart(streamId, ud21x18(0));
}
```

## Recommendation

Modify `_adjustRatePerSecond` to allow setting the same RPS specifically in restart scenario:

```solidity
function _adjustRatePerSecond(uint256 streamId, UD21x18 newRatePerSecond) internal {
    // Allow same RPS if coming from restart
    if (newRatePerSecond.unwrap() == _streams[streamId].ratePerSecond.unwrap() && 
        !isRestartingStream) {
        revert Errors.SablierFlow_RatePerSecondNotDifferent(streamId, newRatePerSecond);
    }
    // Rest of the function...
}
```

Or add specific handling in restart:

```solidity
function _restart(uint256 streamId, UD21x18 ratePerSecond) internal {
    if (_streams[streamId].ratePerSecond.unwrap() != 0) {
        revert Errors.SablierFlow_StreamNotPaused(streamId);
    }
    
    // Skip RPS adjustment if keeping it 0
    if (ratePerSecond.unwrap() != 0) {
        _adjustRatePerSecond(streamId, ratePerSecond);
    }
    
    emit RestartFlowStream(streamId, msg.sender, ratePerSecond);
}
```

## <a id='L-03'></a>L-03. No way to refund max            



## Summary

The Sablier Flow protocol lacks a `refundMax` function, unlike its withdrawal counterpart `withdrawMax`. This asymmetry creates potential issues with transaction failures and front-running opportunities.

## Vulnerability details

The current refund mechanism requires users to:

1. Query refundable amount
2. Submit separate refund transaction

```solidity
function _refund(uint256 streamId, uint128 amount) internal {
    uint128 refundableAmount = _refundableAmountOf(streamId);
    if (amount > refundableAmount) {
        revert Errors.SablierFlow_RefundOverflow(
            streamId, 
            amount, 
            refundableAmount
        );
    }
    // ... rest of refund logic
}
```

## What's the impact?

* Failed transactions due to amount changes between query and execution
* Wasted gas from transaction retries
* Poor UX for senders trying to refund maximum amount
* Increased likelihood of front-running attacks

## What's the likelihood?

LOW - This scenario is likely to occur in active streams where:

* Recipients frequently withdraw
* Senders need to refund maximum amounts
* MEV bots monitor for refund opportunities

## Proof of concept

Let's do a pseudo PoC

```solidity
// Attack Scenario
contract RefundAttack {
    function attack() external {
        // 1. Monitor for refund transactions
        uint128 refundable = flow.refundableAmountOf(streamId);
        
        // 2. Front-run with withdraw
        flow.withdraw(streamId, recipient, amount);
        
        // 3. Original refund transaction fails
        // flow.refund(streamId, refundable); // Reverts
    }
}
```

## Recommendation

Implement `refundMax` function similar to `withdrawMax`:

```solidity
function refundMax(uint256 streamId) external 
    notNull(streamId)
    onlySender(streamId) 
    returns (uint128)
{
    uint128 refundable = _refundableAmountOf(streamId);
    _refund(streamId, refundable);
    return refundable;
}
```




