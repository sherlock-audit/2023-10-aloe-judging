Warm Orange Dragon

high

# No Slippage Protection When Adding and Removing Liquidity and Liquidations
## Summary

The callbacks which add and remove liquidity on Uniswap do not have slippage parameters.

## Vulnerability Detail

When liquidity is added/removed through Aloe's `modify` function, Uniswap's `mint` and `burn` functions are called directly.

The generally correct way to make contract calls is through `increaseLiquidity` and `decreaseLiquidity`. `amount0Min` and `amount1Min` are passed through as slippage parameters which is used when Uniswap is called this way with:

```solidity
nonfungiblePositionManager.decreaseLiquidity(params)
```

With the direct call as implemented in Aloe, attempts to add liquidity can be sandwiched with a transaction which 

1. Pushes the pool price to the wrong price  
2. Allows the victim to add liquidity
3. Sells tokens into the liquidity for a profit.

(different step order and direction for removing liquidiy)

This applies to adding and removing liquidity during `modify`, and the liquidity removal during `liquidate`.

## Impact

- Loss of funds to MEV/Sandwich attacks when adding liquidity
- - Loss of funds to MEV/Sandwich attacks when removing liquidity

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L358-L364

## Tool used

Manual Review

## Recommendation

Call Uniswap's `decreaseLiquidity` function and the corresponding increase function and allow customizable slippage parameters as a user input that are passed down in the `modify` and `liquidate` functions.
