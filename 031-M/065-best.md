Warm Orange Dragon

medium

# Position Can be Opened Which Are Immediately Liquidatable
## Summary

The same check - `isHealthy` - is used both opening positions and liquidating positions, which allows users to open positions which are almost immediately liquidatable.

## Vulnerability Detail

Borrow and Lending platforms usually have a seperate `initial margin`, which is a maximum amount of leverage when opening a position, and `maintenance margin`, which is the threshold after which a position gets liquidated. The two margins are different, with the maintenance margin being higher leverage, to prevent users from getting liquidated almost immediately after opening their position.

Aloe only has a single check - the `isHealthy` check which only ensures that a position cannot be immediately liquidated at the TWAP price during the function call. However, the smallest shift in price will immediately make that position unhealthy and push it into liquidation range.

This applies not only to opening fresh positions, but also to adjusting positions such as increasing liquidity, or decreasing collateral. 

## Impact

Users can be warned and then liquidated almost immediately after opening a position.

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L299-L327

## Tool used

Manual Review

## Recommendation

Implement a seperate initial margin which is seperate and safer than then `isHealthy` check for liquidations
