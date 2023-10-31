Warm Orange Dragon

high

# Uniswap Formula Drastically Underestimates Volatilty
## Summary

The implied volatility calculated fees over a time period divided by current liquidity will almost always be lower than a reasonable derivation of volaitility. This is because there is no incentive or way for rational market participants to "correct" a pool where there is too much liquidity relative to volatility and fees.

## Vulnerability Detail

Note: This report will use annualised IV expressed in % will be use, even though the code representation uses different scaling.

Aloe estimates implied volatility based on the article cited below (taken from in-line code comments)

```solidity

//@notice Estimates implied volatility using this math - https://lambert-guillaume.medium.com/on-chain-volatility-and-uniswap-v3-d031b98143d1).
```

Lambert's article describes a method of valuing Uniswap liquidity positions based on volatility. It is correct to say that the expected value of holding an LP position can be determined by the formula referenced in the article.  A liquidity position can be valued with the same as "selling a straddle" which is a short-volatility strategy which involves selling both a put and a call. Lambert does this by representing fee collection as an options premium and impermanat loss as the cost paid by the seller when the underlying hits the strike price. If the implied volatility of a uniswap position is above the fair IV, then it is profitable to be a liquidity provider, if it is lower, than it is not.

KEY POINT: However, this does not mean that re-arranging the formula to derive IV gives a correct estimation of IV.

The assumptions of the efficient market hypothesis holds true only when there is a mechanism and incentive for rational actors to arbitrage the value of positions to fair value. There is a direct mechanism to push down the IV of Uniswap liquidity positions - if the IV is too high then providing liquidity is +EV, so rational actors would deposit liquidity, and thus the IV as calculated by Aloe's formula will decrease.

However, when the `IV` derived from Uniswap fees and liquidity is too low, there is no mechanism for rational actors to profit off correcting this. If you are not already a liquidity provider, there is no way to provide "negative liquidity" or "short a liquidity position".

In fact the linked article by Lambert Guillaume contains data which demonstrates this exact fact - the table which shows the derived IV at time of writing having far lower results than the historical volatilities and the the IV derived from markets that allow both long and short trading (like options exchanges such as Deribit).  

Here is a quote from that exact article, which points out that the Uniswap derived IV is sometimes 2.5x lower. Also check out the table directly in the article for reference:

```solidity
"The realized volatility of most assets hover between 75% and 200% annualized in ETH terms. If we compare this to the IV extracted from the Uniswap v3 pools, we get:

Note that the volatilities are somewhat lower, perhaps a factor of ~2.5, for most assets."
```


The IV's in options markets or protocols that have long-short mechanisms such as Opyn's Squeeth have a correction mechanism for `IV's` which are too low, because you can both buy and sell options, and are therefore "correct" according to Efficient Market Hypothesis. The Uniswap pool is a "long-only" market, where liquidity can be added, but not shorted, which leads to systematically lower `IV` than is realistic. The EMH model, both in soft and hard form, only holds when there is a mechnaism for a rational minority to profit off correcting a market imbalance. If many irrational or utilitarian users deposits too much liquidity into a Uniswap v3 pool relative to the fee capture and IV, theres no way to profit off correcting this imbalance.

There are 3 ways to validate the claim that the Uniswap formula drastically underestimates the IV:

1. On chain data which shows that the liquidty and fee derivation from Uniswap gives far lower results than other
2. The table provided in Lambert Guillaume's article, which shows a Uniswap pool derived IVs which are far lower than the historical volatilities of the asset.
3. Studies showing that liquidity providers suffer far more impermanent loss than fees.

## Impact

- The lower IV increases LTV, which means far higher LTV for risky assets. `5 sigma` probability bad-debt events, as calculated by the protocol which is basically an impossibility, becomes possible/likely as the relationship between `IV` or `Pr(event)` is super-linear

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/libraries/Volatility.sol#L33-L81

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/VolatilityOracle.sol#L45-L94

## Tool used

Manual Review

## Recommendation

2 possible options (excuse the pun):

- Use historical price differences in the Uniswap pool (similar to a TWAP, but Time Weighted Average Price Difference) and use that to infer volatilty alongside the current implementations which is based on fees and liquidity. Both are inaccurate, but use the `maximum` of the two values. The 2 IV calculations can be used to "sanity check" the other, to correct one which drastically underestimates the risk

- Same as above, use the `maximum`  of the fee/liquidity derived `IV` but use a market that has long/short possibilities such as Opyn's Squeeth to sanity check the IV.