# Liquidity as a service for AMMs - Opal

- 1 high confirmed, 7 medium confirmed and 2 low.

1 - deposit and withdraw functions are missing isActive pool check #365. [Medium]

2 - Missing _`handleRebalancingRewards` in withdrawals, rewards will not be balanced #183. [Medium]

3 - Arithmetic overflow in `changeGaugeWeight` function when weight is less than oldGaugeWeight #136. [Medium]

4 - auraPool.withdrawAndUnwrap is not checking the return value, transaction may fails silently #78. [Medium]

5 - getUSDPrice will revert when access to Chainlink oracle data feed is blocked #31 (Unique). [Medium]

6 - `deployGauge` is missing access control, any one can deploy a malicious lpToken #124. [Medium]

7 - One lpToken can be added to many gauges, breaks the protocol requirements #29. [Medium]

8 - _`exchangeRate` can be manipulated, leads to inflation attack #205 [High].

9 - IBalancerVault.ExitPoolRequest is using minAmountsOut as empty array, allows sandwich attacks #77 ( low/info! ).

10 - Lack of slippage control in _depositToAuraPool, can be front-run #60 (( low/info! )).


### Vulnerability Report: _exchangeRate can be manipulated, leads to inflation attack [HIGH]

**Summary:**
The _exchangeRate in the Omnipool can be manipulated, allowing an attacker to set an arbitrary exchange rate during the deposit process. This manipulation can make it unprofitable for other users to deposit into the pool.

**Vulnerability Exploitation (Proof of Concept):**

1. **Hacker Sets Exchange Rate:**
   - The attacker initiates a deposit by setting the exchangeRate during a deposit.
   - Example:
     ```solidity
     vm.startPrank(hacker);
     omnipool.deposit(2, 0);
     token.transfer(address(omnipool), 2);
     vm.stopPrank();
     ```

2. **Victims Attempt to Deposit:**
   - Other users (victims) try to deposit into the omnipool after the exchange rate manipulation.
   - Due to the manipulated exchange rate, victims receive zero LP tokens for their deposits.
   - Example:
     ```solidity
     vm.startPrank(user);
     token.approve(address(omnipool), 10 ** 18);
     omnipool.deposit(10 ** 18, 0);
     vm.stopPrank();
     ```

3. **Attacker Withdraws All Tokens:**
   - The attacker starts to withdraw all tokens from the pool.
   - Example:
     ```solidity
     vm.startPrank(hacker);
     omnipool.withdraw(1, 0);
     ```

**Impact:**
- The attacker can make the omnipool unprofitable for users by manipulating the exchange rate during deposits, causing victims to receive zero LP tokens.
- The attacker can then withdraw all tokens from the pool, essentially draining it of funds.

**Recommendation:**
One of the simplest solutions is to execute a deposit immediately after the deployment of the contract. By doing so, the exchangeRate can be adjusted to a desired and controlled value, mitigating the risk of potential manipulation.

Some of the recommendations along with their pros and cons, can be found in the [OpenZeppelin github issue](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/3706).
___
