Slow Indigo Woodpecker

high

# Oracle.sol: manipulation via increasing Uniswap V3 pool observationCardinality
## Summary
This issue deals with how the `Oracle.consult` function can be provided with a malicious `seed` such as to return wrong results.  

This is a complex issue that requires multiple steps to be executed in order to set up and execute the attack.  

In depth knowledge of the UniswapV3 `observationCardinality` concept is needed as well as a wholesome understanding of the Aloe II protocol.  

This issue can occur as a result of an intentional attack but also as part of regular operation without attention to attack the protocol (even though unlikely).  

The corrupted data that the `Oracle.consult` function provides as a result of this issue is used upstream by the `VolatilityOracle`.  

The attack path is quite involved. However by exploiting this issue, the TWAP price can be manipulated as well as implied volatility (IV) and the probe prices.

## Vulnerability Detail
The [`Oracle.consult`](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L42-L137) function takes a `uint40 seed` parameter and can be used in either of two ways:  
1. Set the highest 8 bit to a non-zero value [to use Uniswap V3's binary search to get observations](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L51-L56)
2. Set the highest 8 bit to zero and use the lower 32 bits to provide hints and [use the more efficient internal `Oracle.observe` function to get the observations](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L57-L81)

The code for Aloe's [`Oracle.observe`](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L161-L205) function is adapted from Uniswap V3's [Oracle library](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/Oracle.sol).

To understand this issue it is necessary to understand Uniswap V3's `observationCardinality` concept.

A deep dive can be found [here](https://uniswapv3book.com/docs/milestone_5/price-oracle/#observations-and-cardinality).  

In short, it is a circular array of variable size. The size of the array can be increased by ANYONE via calling [`Pool.increaseObservationCardinalityNext`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L255-L267).  

The Uniswap V3 [`Oracle.write`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Oracle.sol#L78-L101) function will then take care of actually expanding the array once the current index has reached the end of the array. 

As can be seen in [this](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Oracle.sol#L108-L120) function, uninitialized entries in the array have their timestamp set to `1`.  

And all other values in the observation struct (array element) are set to zero:  

```solidity
struct Observation {
    // the block timestamp of the observation
    uint32 blockTimestamp;
    // the tick accumulator, i.e. tick * time elapsed since the pool was first initialized
    int56 tickCumulative;
    // the seconds per liquidity, i.e. seconds elapsed / max(1, liquidity) since the pool was first initialized
    uint160 secondsPerLiquidityCumulativeX128;
    // whether or not the observation is initialized
    bool initialized;
}
```

Here's an example for a simplified array to illustrate how the Aloe `Oracle.observe` function might read an invalid value:  

```text
Assume we are looking for the target=10 timestamp.

And the observations array looks like this (element values are timestamps):

| 12 | 20 | 25 | 30 | 1 | 1 | 1 |

The length of the array is 7.

Let's say we provide the index 6 as the seed and the current observationIndex is 3 (i.e. pointing to timestamp 30)

The Oracle.observe function then chooses 1 as the left timestamp and 12 as the right timestamp.

This means the invalid and uninitialized element at index 6 with timestamp 1 will be used to calculate the Oracle values.
```

[Here](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L190-L198) is the section of the `Oracle.observe` function where the invalid element is used to calculate the result.  

By updating the observations (e.g. swaps in the Uniswap pool), an attacker can influence the value that is written on the left of the array, i.e. he can arrange for a scenario such that he can make the Aloe `Oracle` read a wrong value.  

Upstream this causes the Aloe `Oracle` to continue calculation with `tickCumulatives` and `secondsPerLiquidityCumulativeX128s` haing a corrupted value. Either `secondsPerLiquidityCumulativeX128s[0]`, `tickCumulatives[0]` AND `secondsPerLiquidityCumulativeX128s[1]`, `tickCumulatives[1]` or only `secondsPerLiquidityCumulativeX128s[0]`, `tickCumulatives[0]` are assigned invalid values (depending on what the timestamp on the left of the array is).

## Impact
The corrupted values are then used in the [further calculations](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L84-L135) in `Oracle.consult` which reports its results upstream to `VolatilityOracle.update` and `VolatilityOracle.consult`, making their way into the core application.  

The TWAP price can be inflated such that bad debt can be taken on due to inflated valuation of Uniswap V3 liqudity.

Besides that there are virtually endless possibilities for an attacker to exploit this scenario since the Oracle is at the very heart of the Aloe application and it's impossible to foresee all the permutations of values that a determined attacker may use.

E.g. the TWAP price is used for liquidations where an incorrect TWAP price can lead to profit.
If the protocol expects you to exchange 1 BTC for 10k USDC, then you end up with ~20k profit.

Since an attacker can make this scenario occur on purpose by updating the Uniswap observations (e.g. by executing swaps) and increasing observation cardinality, the severity of this finding is "High".  

## Code Snippet
Affected `Oracle.observe` function from Aloe II:
https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L57-L81

`Oracle` library from Uniswap V3 to see how to implement the necessary check for the `initialized` property:
https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/Oracle.sol

## Tool used
Manual Review

## Recommendation
The `Oracle.observe` function must not consider observations as valid that have not been initialized.  

This means the `initialized` field must be queried [here](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L188) and [here](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L171) and must be skipped over.  
