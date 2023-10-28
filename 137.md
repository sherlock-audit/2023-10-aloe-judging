Itchy Glossy Boa

high

# Gamma values are not properly scaled
## Summary

Wrong calculation of gamma affecting wrong computation of pool revenue `computeRevenueGamma`, this means that the computed value may be much lower than it should be, affecting implied volatility on VolatilityOracle update (and its `lastWrites`)

## Vulnerability Detail

There is a PoolMetadata struct defined as follows:

```solidity
File: Volatility.sol
15:     struct PoolMetadata {
16:         // the overall fee minus the protocol fee for token0, times 1e6
17:         uint24 gamma0;
18:         // the overall fee minus the protocol fee for token1, times 1e6
19:         uint24 gamma1;
20:         // the pool tick spacing
21:         int24 tickSpacing;
22:     }
```

According to the struct comments, `gamma0` and `gamma1` represent the difference between the pool fee and the protocol fee, scaled by 1e6. These `gamma` values are crucial for computing pool revenue in the `computeRevenueGamma` function. Specifically, this function is invoked through a series of steps: `VolatilityOracle.update -> Volatility.estimate -> Volatility.computeRevenueGamma`.

The `gamma` values are obtained and utilized exclusively within the `_getPoolMetadata` function, which is called from the `prepare()` function in `VolatilityOracle`. Here's how it is implemented:

```solidity
File: VolatilityOracle.sol
101:     function _getPoolMetadata(IUniswapV3Pool pool) private view returns (Volatility.PoolMetadata memory metadata) {
102:         (, , uint16 observationIndex, uint16 observationCardinality, , uint8 feeProtocol, ) = pool.slot0();
...
111:         uint24 fee = pool.fee();
112:         metadata.gamma0 = fee;
113:         metadata.gamma1 = fee;
114:         unchecked {
115:             if (feeProtocol % 16 != 0) metadata.gamma0 -= fee / (feeProtocol % 16);
116:             if (feeProtocol >> 4 != 0) metadata.gamma1 -= fee / (feeProtocol >> 4);
117:         }
...
120:     }
```

Here, it's clear that `gamma0` and `gamma1` are derived from the pool fee (Lines 111-113) after subtracting the protocol fee (Lines 115-116). However, there's an important aspect missing: the multiplication by `1e6`. 

If we examine the `computeRevenueGamma` function, we observe that it operates on the assumption of a scale factor of `1e6` in its division operation. This discrepancy implies that the result from `computeRevenueGamma` may be understated due to the lack of the `1e6` multiplier in the calculation of `gamma`. This adjustment is crucial to align with the scale specified in the struct's comment and the `1e6` scale factor used in the division operation within `computeRevenueGamma`.

```solidity
File: Volatility.sol
103:     function computeRevenueGamma(
104:         uint256 feeGrowthGlobalAX128,
105:         uint256 feeGrowthGlobalBX128,
106:         uint160 secondsPerLiquidityX128,
107:         uint32 secondsAgo,
108:         uint24 gamma
109:     ) internal pure returns (uint256) {
110:         unchecked {
111:             uint256 delta;
112:
113:             if (feeGrowthGlobalBX128 >= feeGrowthGlobalAX128) {
114:                 // feeGrowthGlobal has increased from time A to time B
115:                 delta = feeGrowthGlobalBX128 - feeGrowthGlobalAX128;
116:             } else {
117:                 // feeGrowthGlobal has overflowed between time A and time B
118:                 delta = type(uint256).max - feeGrowthGlobalAX128 + feeGrowthGlobalBX128;
119:             }
120:
121:             return Math.mulDiv(delta, secondsAgo * uint256(gamma), secondsPerLiquidityX128 * uint256(1e6));
122:         }
123:     }
```

## Impact

the computed value in `computeRevenueGamma` may be much lower than it should be due to huge scale difference

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/VolatilityOracle.sol#L112-L117

## Tool used

Manual Review

## Recommendation

to ensure correct calculations, gamma should be multiplied by 1e6 when computing it in `_getPoolMetadata`. This will ensure that the subsequent calculations in computeRevenueGamma are performed with the correct scale.
