# First Flight #6: Voting Booth - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Critical Bug: Incorrect Rewards Distribution Logic](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Compiler Version Incompatibility with Layer 2 Networks: Deployment Issues on Arbitrum](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #6

### Dates: Dec 15th, 2023 - Dec 22nd, 2023

[See more contest details here](https://www.codehawks.com/contests/clq5cx9x60001kd8vrc01dirq)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 1
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Critical Bug: Incorrect Rewards Distribution Logic            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L192

## Summary

A critical bug has been identified in the rewards distribution logic of the smart contract. The issue arises when the proposal is approved, and rewards are intended to be distributed among the "for" voters. However, the current implementation incorrectly uses the total number of voters (both "for" and "against") as the denominator in the reward calculation, leading to improper distribution and potential fund locking within the contract.

## Vulnerability Details

### Code Location
The issue is located in the `distributeRewards` function within the smart contract.

### Vulnerability Description
The rewards distribution logic incorrectly calculates the denominator for the reward distribution. Instead of considering only the "for" voters, it uses the total number of voters (both "for" and "against"). This leads to a situation where the rewards are distributed among a larger group than intended, causing incorrect fund allocation.

### Code snippet
```solidity
// Incorrect calculation of totalVotes in distributeRewards function
uint256 totalVotesFor = s_votersFor.length;
uint256 totalVotesAgainst = s_votersAgainst.length;
uint256 totalVotes = totalVotesFor + totalVotesAgainst;

// Incorrect usage of totalVotes in rewardPerVoter calculation
uint256 rewardPerVoter = totalRewards / totalVotes;
```

### Impact
The impact of this vulnerability is severe. It results in an incorrect distribution of rewards, causing funds to be allocated to a larger group of voters than intended. This can lead to fund locking within the contract and potential financial losses for users.

## POC:

- Copy the below test and add it in VotingBoothTest.t.sol
- run this command `forge test --match-test testVotePassesAndMoneyIsNotSentToAllVoters -vvvv`
- you will get the below result

**Result:**
```
Running 1 test for test/VotingBoothTest.t.sol:VotingBoothTest
[PASS] testVotePassesAndMoneyIsNotSentToAllVoters() (gas: 357838)
Logs:
  Balance 1 Before:  0
  Balance 2 Before:  0
  Balance 3 Before:  0
  Balance 4 Before:  0
  Balance 5 Before:  0
  Balance 1 After:  2000000000000000000
  Balance 2 After:  0
  Balance 3 After:  0
  Balance 4 After:  2000000000000000000
  Balance 5 After:  2000000000000000000
```

**Test Code:**

```solidity
function testVotePassesAndMoneyIsNotSentToAllVoters() public {
        vm.prank(address(0x1));
        booth.vote(true);
        console.log("Balance 1 Before: ", address(0x1).balance);

        vm.prank(address(0x2));
        booth.vote(false);
        console.log("Balance 2 Before: ", address(0x2).balance);

        vm.prank(address(0x3));
        booth.vote(false);
        console.log("Balance 3 Before: ", address(0x3).balance);

        vm.prank(address(0x4));
        booth.vote(true);
        console.log("Balance 4 Before: ", address(0x4).balance);

        vm.prank(address(0x5));
        booth.vote(true);
        console.log("Balance 5 Before: ", address(0x5).balance);

        console.log("Balance 1 After: ", address(0x1).balance);
        console.log("Balance 2 After: ", address(0x2).balance);
        console.log("Balance 3 After: ", address(0x3).balance);
        console.log("Balance 4 After: ", address(0x4).balance);
        console.log("Balance 5 After: ", address(0x5).balance);

        assert(!booth.isActive() && address(booth).balance > 0);
}
```

## Tools Used
- Manual review + Fuzzing via foundry.

## Recommendations

 **Update distributeRewards Function**: Modify the `distributeRewards` function to correctly calculate the total number of "for" voters and use this value as the denominator in the reward distribution calculation.

```solidity
    // Correct calculation of totalVotesFor in distributeRewards function
    uint256 totalVotesFor = s_votersFor.length;

    // Correct usage of totalVotesFor in rewardPerVoter calculation
    uint256 rewardPerVoter = totalRewards / totalVotesFor;
```

Also remove the round up check, if this is NOT removed the last voter might receive less reward than others.

```diff
-if (i == totalVotesFor - 1) {
-    rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);
-}
```


		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Compiler Version Incompatibility with Layer 2 Networks: Deployment Issues on Arbitrum            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L2

### **Summary**
The protocol employs Solidity compiler version 0.8.23, yet it aims for deployment on Arbitrum, a Layer 2 (L2) network. However, this introduces bytecode complications involving the PUSH0 opcode, which is incompatible with Arbitrum. The recommended solution is to either revert to Solidity compiler version 0.8.19 or incorporate `evm_version = "paris"` in the `foundry.toml` configuration file to ensure compatibility with Arbitrum.

### **Vulnerability Details**
The issue lies in the bytecode generated by Solidity compiler version 0.8.23, specifically with the PUSH0 opcode, causing deployment failures on L2 networks like Arbitrum. This arises due to the lack of support for PUSH0 on certain L2 networks.

### **Impact**
Deployment failures on Arbitrum or other L2 networks pose a significant hindrance to the protocol's successful launch. The incompatibility may affect the functionality and accessibility of the protocol on these networks.

### **Tools Used**
- Manual Review

### **Recommendations**
To address the problem, the recommended course of action is to downgrade the Solidity compiler version to 0.8.19 or insert `evm_version = "paris"` in the `foundry.toml` configuration file. This ensures alignment with L2 networks like Arbitrum, avoiding deployment failures and ensuring the protocol's compatibility.






