# Liquidity as a service for AMMs - Opal

- 1 high confirmed, 7 medium confirmed and 2 low but let's see

1 - deposit and withdraw functions are missing isActive pool check #365
2 - Missing _`handleRebalancingRewards` in withdrawals, rewards will not be balanced #183
3 - Arithmetic overflow in `changeGaugeWeight` function when weight is less than oldGaugeWeight #136 (this is valid one but not marked me, refer to escalations)
4 - auraPool.withdrawAndUnwrap is not checking the return value, transaction may fails silently #78
5 - getUSDPrice will revert when access to Chainlink oracle data feed is blocked #31 (Unique)
6 - `deployGauge` is missing access control, any one can deploy a malicious lpToken #124
7 - One lpToken can be added to many gauges, breaks the protocol requirements #29
8 - _`exchangeRate` can be manipulated, leads to inflation attack #205 [High]
9 - IBalancerVault.ExitPoolRequest is using minAmountsOut as empty array, allows sandwich attacks #77 ( low/info! )
10 - Lack of slippage control in _depositToAuraPool, can be front-run #60 (( low/info! ))