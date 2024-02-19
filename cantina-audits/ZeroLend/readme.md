# Title: Expiration Check Missing in ZeroLocker's Merge Function
- Status: Confirmed
- Severity: Medium
- Likelihood: Medium
- Impact: Medium
[Cantina link](https://cantina.xyz/code/a83eaf73-9cbc-495f-8607-e55d4fdaf407/findings/6f30b930-3264-4046-9b46-1ba8c48604a1?q=status%253Adisputed%252Crejected%252Cduplicate%252Cpotentially_duplicate%252Cconfirmed%252Cacknowledged%252Cfixed%252Cescalated%2520reviewers%253A0xTheBlackPanther)

## **Summary:**
The merge function in the ZeroLocker contract has a potential issue as it lacks explicit checks for the expiration status of the source lock (_from) and destination lock (_to) before attempting the merge operation. This oversight could lead to unexpected behavior, especially when dealing with expired locks, and poses a risk to the security and functionality of the merging operation.

## **Vulnerability Details:**
The vulnerability arises due to the absence of checks for the expiration status of the source and destination locks in the merge function. Without proper verification, the function may allow the merging of expired locks, leading to undesired outcomes.

## **Impact:**
The impact of this vulnerability is primarily related to unexpected behavior during the merging process. Merging expired locks is allowed, contrary to the intended behavior, potentially compromising the security and reliability of the ZeroLocker contract.

## **POC:**
```solidity
// Copy below test and run it via cmd forge test --match-test testMergeWithExpireLocks -vvvv
function testMergeWithExpireLocks() public {
    // ... (Test setup details)
    // now let's try merging
    zeroLocker.merge(bobTokenIdFrom, aliceTokenIdTo);
}
```
Result:
```
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 14.97ms
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## **Recommendations:**
To address this potential issue, it is recommended to implement checks for the expiration status of both the source lock (_from) and destination lock (_to) before proceeding with the merge operation. These checks should ensure that expired locks are handled appropriately, aligning with the contract's logic and the desired behavior for merging expired locks.

## **Modified Merge Function:**
```solidity
function merge(uint256 _from, uint256 _to) external override {
    // ... (Existing checks)
    // Check if the source lock (_from) is not expired
    require(_locked0.end > block.timestamp, "source lock expired");
    // Check if the source lock (_to) is not expired
    require(_locked1.end > block.timestamp, "destination lock expired");
    // ... (Existing logic)
}
```

*Note: The modified merge function includes additional checks for the expiration status of the source and destination locks.*