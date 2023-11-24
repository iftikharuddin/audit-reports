# Checklist before auditing protocols

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

## Bridges DeFi Attacks

## Oracles

- [M-01] The Chainlink price feed's input is not properly validated ( check for stale and negative returns )