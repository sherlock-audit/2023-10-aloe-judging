Rare Violet Caribou

high

# Users positions can't be liquidated since protocol is using Use of slot0 to get sqrtPriceLimitX96
## Summary
Users positions can't be liquidated since protocol is using Use of slot0 to get sqrtPriceLimitX96
## Vulnerability Detail
In the `Borrower.sol` the function liquidate() Fetch prices from oracle using 
```solidity
 (Prices memory prices, ) = getPrices(oracleSeed);
```
 and the function `getPrices()` is using an internal function `_getPrices()` to fetch the price
 
```solidity
 function getPrices(uint40 oracleSeed) public view returns (Prices memory prices, bool seemsLegit) {
        (, uint8 nSigma, uint8 manipulationThresholdDivisor, ) = FACTORY.getParameters(UNISWAP_POOL);
        (prices, seemsLegit) = _getPrices(oracleSeed, nSigma, manipulationThresholdDivisor);
    }
```
further the internal function `_getPrices()` is calling `Oracle.consult()`

```solidity
 (metric, prices.c, iv) = ORACLE.consult(UNISWAP_POOL, oracleSeed);
```
 and the `PoolData` is stored by calling another internal function `consult()` given in the code below :
 https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/libraries/Oracle.sol#L45
 
 the problem lies within the line 

```solidity
 (data.sqrtPriceX96, data.currentTick, observationIndex, observationCardinality, , , ) = pool.slot0();
```
 
 where the current sqrtPriceLimitX96 is retrieved from the `pool.slot0()`, however the `sqrtPriceX96` gotten from `pool.slot0` is the most recent data point and can be manipulated easily via MEV bots & Flashloans with sandwich attacks and  can cause lose of funds when interacting with the Uniswap.swap function.
 
## Impact
 The `sqrtPriceX96` gotten from pool.slot0 is not correct and an Attacker can Simply manipulate the `sqrtPriceX96` and if the `Uniswap.swap` function is called with the `sqrtPriceX96` the token will be bought at a higher price, and The Attacker would back run the transaction to sell thereby making gain but causing loss to whoever called those functions.
## Code Snippet
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/libraries/Oracle.sol#L45
## Tool used

Manual Review

## Recommendation
Use The `TWAP` to get the value of `sqrtPriceX96`.

[here](https://github.com/charmfinance/alpha-vaults-contracts/blob/07db2b213315eea8182427be4ea51219003b8c1a/contracts/AlphaStrategy.sol#L136-L144) is an example of TWAP implementation