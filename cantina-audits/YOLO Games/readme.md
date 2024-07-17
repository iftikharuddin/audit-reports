## YOLO Games

[YOLO Games](https://cantina.xyz/competitions/a2c3cc6a-e384-495f-9751-5d7e657bc219) is the hub of degen gaming. Play and earn across a range of provably fair, on-chain games.

Ranked [6](https://cantina.xyz/competitions/a2c3cc6a-e384-495f-9751-5d7e657bc219/leaderboard) in this by finding just one medium ;) Finding details below:

## Summary

The `setFeeSplit` function allows the owner to set the fee structure for games, including the protocol fee and the liquidity pool fee. This function is only callable by the contract owner and enforces that the combined fees do not exceed 5%. However, it lacks protections to prevent the owner from changing the fee configuration during an ongoing game, which can lead to higher fees being applied retroactively to players without their knowledge.

## Impact

Allowing the admin to change the fee structure mid-game can unfairly increase the fees charged to players, exceeding the fees they agreed to at the start of the game. For instance, in a game where players originally agreed to a combined fee of 2%, the admin can change this to 4% while the game is live. When the player cashes out, they are charged the new higher fee, resulting in a loss of additional funds. This not only affects the player's financial outcomes but also undermines trust in the game's fairness and integrity.

## Proof of Concept

1. A player starts a game with the following fee configuration: 1% protocol fee and 1% liquidity pool fee, totaling a combined fee of 2%.
    ```solidity
    setFeeSplit(gameAddress, 100, 100); // 1% protocol fee and 1% liquidity pool fee
    ```

2. During the game, the admin updates the fee configuration to: 2% protocol fee and 2% liquidity pool fee, totaling a combined fee of 4%.
    ```solidity
    setFeeSplit(gameAddress, 200, 200); // 2% protocol fee and 2% liquidity pool fee
    ```

3. The player, unaware of the fee change, continues to play and eventually cashes out. The fees calculated during cashout use the updated configuration, charging 4% instead of the original 2%.

    ```solidity
    IGameConfigurationManager.FeeSplit memory feeSplit = GAME_CONFIGURATION_MANAGER.getFeeSplit(address(this));
    Fee memory fee = Fee({
        protocolFee: (game.params.playAmountPerRound * multiplier * feeSplit.protocolFeeBasisPoints) / 1e8,
        liquidityPoolFee: (game.params.playAmountPerRound * multiplier * feeSplit.liquidityPoolFeeBasisPoints) / 1e8
    });
    ```

4. The player is charged more than the initially agreed-upon fees, leading to unexpected financial losses.

## Recommendation

Modify the `setFeeSplit` function to check if the game is currently active. If the game is live, do not allow any changes to the fee structure.

