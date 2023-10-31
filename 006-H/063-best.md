Warm Orange Dragon

high

# IV Can be Decreased for Free
## Summary

The liquidity parameter used to calculate IV costs nothing to massively manipulate upwards and doesn't require a massive amount of capital. This makes IV easy to manipulate downwards.

## Vulnerability Detail

The liquidity at a single `tickSpacing` is used to calcualte the `IV`. The more liquidity is in this tick spacing, the lower the `IV`, as demonstarated by the `tickTvl` dividing the return value of the `estimate` function:

```solidity
            return SoladyMath.sqrt((4e24 * volumeGamma0Gamma1 * scale) / (b.timestamp - a.timestamp) / tickTvl);
```

Since this is using data only from the block that the function is called, the liuquidyt can easily be increased by: 

1. depositing a large amount liquidity into the `tickSpacing`
2. calling update
3. removing the liquidity

Note that only a small portion of the total liquidity is in the entire pool is in the active liquidity tick. Therefore, the capital cost required to massively increase the liquidity is low. Additionally, the manipulation has zero cost (aside from gas fees), as no trading is done through the pool. Contract this with a pool price manipulation, which costs a significant amount of trading fees to trade through a large amount of the liquidity of the pool.

Since this manipulation costs nothing except gas, the `IV_CHANGE_PER_UPDATE` which limits of the amount that IV can be manipulated per update does not sufficiently disincentivise manipulation, it just extends the time period required to manipulate.

Decreasing the IV increases the LTV, and due to the free cost, its reasonable to increase the LTV to the max LTV of 90% even for very volatile assets. Aloe uses the IV to estimate the probability of insolvency of loans. With the delay inherent in TWAP oracle and the liquidation delay by the `warn`-then-liquidate process, this manipulation can turn price change based insolvency from a 5 sigma event (as designed by the protocol) to a likely event.

## Impact

- Decreasing IV can be done at zero cost aside from gas fees. 
- This can be used to borrow assets at far more leverage than the proper LTV
- Borrowers can use this to avoid liquidation
- This also breaks the insolvency estimation based on IV for riskiness of price-change caused insolvency.

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/libraries/Volatility.sol#L44-L81

## Tool used

Manual Review

## Recommendation

Use the time weighted average liquidity of in-range ticks of the recent past, so that single block + single tickSpacing liquidity deposits cannot manipulate IV significantly.
