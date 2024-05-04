# Beanstalk Part 2 - Findings Report

- ## Medium Risk Findings
    - ### [M-02. Lack of else condition in `getWstethEthPrice`, no value will be return](#M-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Beanstalk

### Dates: Apr 1st, 2024 - Apr 15th, 2024

[See more contest details here](https://www.codehawks.com/contests/clu7665bs0001fmt5yahc8tyh)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - Medium: 1 Valid medium
		
# Medium Risk Findings
 
## <a id='M-02'></a>M-02. Lack of else condition in `getWstethEthPrice`, no value will be return            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-beanstalk-2/blob/27ff8c87c9164c1fbff054be5f22e56f86cdf127/protocol/contracts/libraries/Oracle/LibWstethEthOracle.sol#L68-L98


## Summary

There is an issue in the `getWstethEthPrice` function, the absence of an `else` condition at the end of the `if` statement block will lead to returning nothing at the end. In simple words, this means that if the condition within the `if` statement (`LibOracleHelpers.getPercentDifference(chainlinkPrice, uniswapPrice) < MAX_DIFFERENCE`) evaluates to false, the function will exit without returning a value.

## Impact

Without an `else` condition, the function may terminate abruptly without providing any meaningful output if the condition in the `if` statement is not met. This can lead to unpredictable behavior, making it difficult to determine why the function failed to produce a result.

For example if someone calls `getWstethEthPrice` or this is called internally in a function for returning the `wstETH/USD` price with the option of using a TWA lookback. And the `if` condition fails so this will return no value to the user or any function calling it will get no value in return.

## Recommendation

Add an `else` condition at the end of the `if` statement block to handle cases where the condition is not met. This could involve returning a default value or triggering an error to indicate that the function was unable to compute a valid price.

For example, new code will look like this:

```solidity
if (LibOracleHelpers.getPercentDifference(chainlinkPrice, uniswapPrice) < MAX_DIFFERENCE) {
    // Calculation logic...
} else {
    // Handle case where condition is not met
    // For example, return a default value or trigger an error
    return 0; // Or trigger a revert with an error message
}
```



