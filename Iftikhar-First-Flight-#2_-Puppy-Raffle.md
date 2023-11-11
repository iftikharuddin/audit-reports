# First Flight #2: Puppy Raffle - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. use transfer instead of sendValue, potential reentrancy attacks](#H-01)
    - ### [H-02. current randmoness is predictable, and can be manipulated](#H-02)
    - ### [H-03. withdrawFees has no access control, anyone can withdraw the fees](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Gas-Intensive Duplicate Check in `PuppyRaffle::enterRaffle` Potentially Prone to Denial-of-Service Attacks](#M-01)
- ## Low Risk Findings
    - ### [L-01. Zero Return, can confuse users & devs](#L-01)
    - ### [L-02. use ERC721A instead of ERC721 for gas efficiency and security](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #2

### Dates: Oct 25th, 2023 - Nov 1st, 2023

[See more contest details here](https://www.codehawks.com/contests/clo383y5c000jjx087qrkbrj8)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 1
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. use transfer instead of sendValue, potential reentrancy attacks            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L101

## Summary

Use the built-in `transfer` method for secure and safe Ether transfers in Solidity, as it automatically reverts the transaction in case of failure, preventing unintended loss of funds and enhancing contract security

## Vulnerability Details

The use of the `sendValue` method for Ether transfers can potentially expose the contract to security vulnerabilities due to its lack of automatic transaction reversal on failure, making it susceptible to reentrancy attacks and unintentional fund loss; using the secure `transfer` method is recommended.

## Impact

The vulnerability could lead to reentrancy attacks, fund losses, and overall security risks, highlighting the importance of replacing `sendValue` with the more secure `transfer` method for Ether transfers in the contract.

## Tools Used

- Foundry and manual review

## Recommendations

"Using `transfer` instead of `sendValue` is a security best practice, as it automatically reverts the transaction in case of failure, preventing potential Ether loss due to errors in the recipient contract, thus enhancing contract security."

```diff
-payable(msg.sender).sendValue(entranceFee);
+payable(msg.sender).transfer(entranceFee);
```
    
## <a id='H-02'></a>H-02. current randmoness is predictable, and can be manipulated            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L128

## Summary

The contract relies on on-chain data for randomness, which can be predictable and subject to manipulation, raising concerns about the fairness of winner selection

## Vulnerability Details

The contract employs the `keccak256` function for generating randomness to select a winner, which, in an on-chain context, can be predictable and potentially manipulated by malicious actors. This predictability undermines the integrity of the selection process, potentially allowing unfair advantages or gaming of the system

## Impact

The predictable and manipulable randomness mechanism in the contract can lead to several adverse consequences. It may allow malicious users to influence the selection of winners, thereby compromising the fairness and integrity of the raffle. This impact could result in a loss of trust and reputation for the contract and its operators, as well as potential financial losses for participants.

## Tools Used

- manual review & foundry

## Recommendations

Consider implementing a more secure source of randomness like external oracles or Chainlink VRF for improved raffle fairness and security.

You can implement **Chainlink VRF** in the `selectWinner` function to replace the current randomness generation. Here's an example of how you can modify the selectWinner function to use Chainlink VRF for generating random numbers:

Steps:

1 - Import VRFConsumerBase `import "@chainlink/contracts/src/v0.8/VRFConsumerBase.sol";`
2 - Import VRFConsumerBase to PuppyRaffle contract
```diff
-contract PuppyRaffle is ERC721, Ownable {
+contract PuppyRaffle is ERC721, Ownable, VRFConsumerBase { // Import VRFConsumerBase
```
3 - Declare these vars and add the below to constructor
```
    uint256 public randomResult; // Store the random number

    // Constructor with Chainlink VRF configuration
    constructor(
        uint256 _entranceFee,
        address _feeAddress,
        uint256 _raffleDuration,
        address _vrfCoordinator,
        address _link,
        bytes32 _keyHash,
        uint256 _fee
    ) ERC721("Puppy Raffle", "PR") VRFConsumerBase(_vrfCoordinator, _link) {
        entranceFee = _entranceFee;
        feeAddress = _feeAddress;
        raffleDuration = _raffleDuration;
        raffleStartTime = block.timestamp;

        // ... Other constructor code ...

        keyHash = _keyHash; // Chainlink VRF key hash
        fee = _fee; // Chainlink VRF fee
    }
```

4 - And now add the below function for fair randomness

```
function selectWinner() external {
    require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
    require(players.length >= 4, "PuppyRaffle: Need at least 4 players");

    // Request random number from Chainlink VRF
    requestRandomNumber();

    // The actual winner selection will occur in fulfillRandomness function
}

// Chainlink VRF callback to handle random number
function fulfillRandomness(bytes32 requestId, uint256 randomness) internal override {
    require(msg.sender == vrfCoordinator, "Fulfillment only allowed by Coordinator");
    randomResult = randomness;

    // Use the randomResult for winner selection
    uint256 winnerIndex = randomResult % players.length;
    address winner = players[winnerIndex];
    uint256 totalAmountCollected = players.length * entranceFee;
    uint256 prizePool = (totalAmountCollected * 80) / 100;
    uint256 fee = (totalAmountCollected * 20) / 100;
    totalFees = totalFees + uint64(fee);

    uint256 tokenId = totalSupply();

    // We use a different RNG calculate from the winnerIndex to determine rarity
    uint256 rarity = uint256(keccak256(abi.encodePacked(msg.sender, block.difficulty))) % 100;
    if (rarity <= COMMON_RARITY) tokenIdToRarity[tokenId] = COMMON_RARITY;
    else if (rarity <= COMMON_RARITY + RARE_RARITY) tokenIdToRarity[tokenId] = RARE_RARITY;
    else tokenIdToRarity[tokenId] = LEGENDARY_RARITY;

    delete players;
    raffleStartTime = block.timestamp;
    previousWinner = winner;
    (bool success,) = winner.call{value: prizePool}("");
    require(success, "PuppyRaffle: Failed to send prize pool to winner");
    _safeMint(winner, tokenId);
}

// ... Other contract functions ...

// Function to request a random number from Chainlink VRF
function requestRandomNumber() private {
    require(LINK.balanceOf(address(this)) >= fee, "Not enough LINK to fulfill request");
    bytes32 requestId = requestRandomness(keyHash, fee);
    // You can store the requestId for reference if needed
}

```

Make sure to add test `LINK`

## <a id='H-03'></a>H-03. withdrawFees has no access control, anyone can withdraw the fees            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L157

## Summary

The vulnerability of missing access control in the "withdrawFees" function allows unauthorized users to potentially withdraw fees, leading to the risk of fund loss and financial disruption for the contract.

## Vulnerability Details

`PuppyRaffle:withdrawFees` function lacks access control, allowing anyone to withdraw the fees

## Impact

Unauthorized individuals can withdraw the fees, potentially resulting in the loss of collected funds intended for the owner and disrupting the contract's financial operations.

## Tools Used

- Manual review & foundry

## Recommendations

```diff
-function withdrawFees() external {
+function withdrawFees() external onlyOwner {
```

Make sure to add the modifier code:

```
modifier onlyOwner() {
    require(msg.sender == owner, "Only the contract owner can call this function");
    _;
}
```

		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Gas-Intensive Duplicate Check in `PuppyRaffle::enterRaffle` Potentially Prone to Denial-of-Service Attacks            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L86

## Summary

Gas-Intensive Duplicate Check in `PuppyRaffle::enterRaffle` Potentially Prone to Denial-of-Service Attacks, Resulting in Increased Gas Costs for Participants When Entering the Raffle.

## Vulnerability Details

The function `PuppyRaffle::enterRaffle` employs a loop to inspect the `players` array for duplicate entries. As the `players` array grows in size, the number of checks required for a new player increases proportionally. Consequently, participants who enter the raffle immediately upon commencement will incur significantly lower gas costs compared to those who join at a later stage. Each additional address introduced to the `players` array compounds the computational load of the loop. This differential in gas consumption between early and late entrants raises concerns of unequal participation costs and a potential susceptibility to denial-of-service attacks.

## Impact

This vulnerability results in an unfair raffle experience for users, where participants who enter the raffle at different times face unequal gas costs. This inequality undermines the fairness of the raffle, potentially discouraging late entrants and favoring those who participate early.

## Tools Used

- Foundry
- Manual review

## Proof of Concept

To substantiate the identified vulnerability, a test case was developed to demonstrate its practical implications. The test case, `testCheckForDuplicates`, exemplifies how the nested loop within the `enterRaffle` function creates a potential Denial-of-Service (DoS) vector attack. Furthermore, it highlights the disparity in gas fees between participants who enter the raffle early versus those who join later.

In the test case, the following actions are performed:

1. An array of 100 player addresses is generated and used to simulate the entry of the initial 100 participants into the raffle.

2. The gas consumption for these initial participants is measured, representing the gas cost incurred by early entrants.

3. An additional 5 players are introduced, symbolizing later entrants who join the raffle after its initiation.

4. The gas usage for these late entrants is measured to reflect the gas cost faced by participants who enter at a later stage.

The test case conclusively demonstrates that early participants encounter significantly lower gas costs compared to later entrants. The evidence provided through the test case underscores the vulnerability's impact on gas efficiency and fairness in the raffle, serving as a tangible illustration of the issue's real-world consequences.

testCheckForDuplicates:

```
function testCheckForDuplicates() public {
    vm.txGasPrice(1);

    uint256 playersNum = 100;
    address[] memory players = new address[](playersNum);

    for (uint256 i = 0; i < playersNum; i++) {
        players[i] = address(i);
    }

    // see how much gas it cost to enter
    uint256 gasStart = gasleft();
    puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players);
    uint256 gasEnd = gasleft();
    uint256 gasUsedFirst = (gasStart - gasEnd) * tx.gasprice;
    console.log("Gas cost of the first 100 Players", gasUsedFirst);

    // enter 5 more players
    for (uint256 i = 0; i < playersNum; i++) {
        players[i] = address(i + playersNum);
    }

    // lets see how much gas it cost to enter now for later players
    gasStart = gasleft();
    puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players);
    gasEnd = gasleft();
    uint256 gasUsedSecond = (gasStart - gasEnd) * tx.gasprice;
    console.log("Gas Cost of the 2nd 100 players", gasUsedSecond);

    assert(gasUsedFirst < gasUsedSecond);
//   Logs:
//     Gas cost of the first 100 Players 6252039
//     Gas Cost of the 2nd 100 players 18067748

}
```

## Recommendations

To mitigate this vulnerability and improve gas efficiency, consider using a different data structure to store and track participants. One common approach is to use a mapping to keep track of participants' uniqueness. Here's an example of how you can refactor the code:

```solidity
contract PuppyRaffle {
    // Use a mapping to track participants
    mapping(address => bool) public isParticipant;

    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");

        for (uint256 i = 0; i < newPlayers.length; i++) {
            address player = newPlayers[i];

            // Check if the player is already a participant
            require(!isParticipant[player], "PuppyRaffle: Duplicate player");

            // Mark the player as a participant
            isParticipant[player] = true;
        }

        emit RaffleEnter(newPlayers);
    }
}
```

In this refactored code, a mapping called `isParticipant` is used to keep track of whether an address has already participated in the raffle. When a player enters the raffle, the code checks if the player is already a participant using the `isParticipant` mapping. If the player is not already a participant, they are marked as a participant, and the `isParticipant` mapping is updated.

This approach eliminates the need for nested loops to check for duplicates, making the code more efficient and secure against DoS attacks, as gas costs are now predictable and independent of the number of participants.

# Low Risk Findings

## <a id='L-01'></a>L-01. Zero Return, can confuse users & devs            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L116

## Summary

To improve clarity and avoid potential confusion, it's advisable to change the return value of the `getActivePlayerIndex` function from 0 to -1 or use a more descriptive constant value when the player is not found in the array. This adjustment ensures that a valid index (0) in the array does not mislead users into thinking that the player was found.

## Vulnerability Details

Returning 0 in the `getActivePlayerIndex` function may cause confusion, as it's a valid array index, potentially leading to misinterpretation by users. It's advisable to use a different return value, like -1, to clearly indicate the absence of the player and avoid such confusion.

## Impact

Returning 0 in the `getActivePlayerIndex` function might mislead users into thinking that 0 corresponds to a valid index in the array. This could potentially lead to incorrect interpretations of the function's output and incorrect usage by developers.

## Tools Used

- Foundry, Etherscan, and manual review

## Recommendations

To improve the clarity and avoid confusion, you can use a different constant value, such as `-1`, to indicate that the player was not found in the array. Here's a modified version of the `getActivePlayerIndex` function:

```solidity
function getActivePlayerIndex(address player) external view returns (int256) {
    for (int256 i = 0; i < int256(players.length); i++) {
        if (players[uint256(i)] == player) {
            return i;
        }
    }
    return -1; // Use -1 to indicate that the player was not found
}
```

This updated function returns `-1` if the player is not found, making it clearer that the player was not present in the array.
## <a id='L-02'></a>L-02. use ERC721A instead of ERC721 for gas efficiency and security            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L18

## Summary

Exploring the usage of ERC721A over ERC721 due to its notable advantages in gas efficiency and security optimization. This change can potentially enhance the performance and robustness of the contract.

## Vulnerability Details

Using the standard ERC721 token standard in smart contracts can expose vulnerabilities, including high gas costs, reentrant attacks, front-running, and potential issues related to approval management, integer arithmetic, and metadata handling

## Impact

Utilizing the standard ERC721 can lead to substantial gas costs and a lack of security optimization.

## Tools Used

- Foundry
- Manual review

## Recommendations

Consider using ERC721A to mitigate gas costs and enhance security in your contract.

```diff
-contract PuppyRaffle is ERC721, Ownable {
+contract PuppyRaffle is ERC721A, Ownable {
```

- Make sure to import ERC721A ( https://github.com/chiru-labs/ERC721A )




 


