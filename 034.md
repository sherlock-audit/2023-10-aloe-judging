Loud Cloud Salamander

medium

# Some values, such as `LIQUIDATION_INCENTIVE`, are fixed constants, while the volatility is variable and different between different pools and at different times, making it unprofitable for liquidators to liquidate in certain situations
## Summary

Some of the protocol constants are fixed value constants for all markets at all times. However, since volatility is different in different markets and at different times, there might be situations when high volatility will make it unprofitable for liquidators to liquidate accounts, because the liquidation prices are lagging (as they're averages over somewhat long periods of times). The `LIQUIDATION_INCENTIVE`, for example, which is currently set to 20 (5%), might not be enough to compensate lagging liquidation prices in some cases. This will lead to accounts not being liquidated in time due to liquidations being unprofitable in some situations and large bad debt being generated in the protocol, which can cause bank run and loss of funds for the other protocol users.

Some of the constants which are fixed, but should depend on market/volatility:
- `LIQUIDATION_INCENTIVE`
- `UNISWAP_AVG_WINDOW`
- `LIQUIDATION_GRACE_PERIOD`
- `MAX_LEVERAGE`

## Vulnerability Detail

When account is liquidated and assets swap is required, the liquidator is required to swap account assets at current average pool price over the last `UNISWAP_AVG_WINDOW` (currently 30 minutes) time period plus bonus of the `LIQUIDATION_INCENTIVE` (currently 5%). Additionally, liquidator first has to warn the user and wait for `LIQUIDATION_GRACE_PERIOD` (currently 2 minutes) before liquidating.

The account health does depend on current volatility. However, once it's determined that the account is not healthy, the liquidation itself happens at that average price over the last 30 minutes, which can be unprofitable for liquidator for very long periods of time. For example, if any token price starts falling steadily for some reason (which has happened multiple times in the past), the average price over the last 30 minutes can stay 5%+ away from current price making liquidations unprofitable until the difference is less than 5% (`LIQUIDATION_INCENTIVE` is not enough to make it profitable for liquidator). Or some highly volatile token might often have periods of strong moves either up or down, causing average price to be more than 5% away from current price and making liquidations unprofitable.

Example scenario of user going into bad debt due to lack of liquidations (due to unprofitable liquidations):
1. Some token (TKN) starts falling from the price of $100 to $90
2. Current price is $90, average over the last 30 minutes is $95
3. Alice account (assets = 1.2 TKN, debt = 95 USDT) becomes unhealthy
4. Bob tries to liquidate Alice account, however in order to liquidate, he has to swap 1 TKN into 95 USDT (essentially buying 1 TKN for 95 USDT). His liquidation incentive for this is 5% or 95 * 5% = $4.75. This means he has to buy 1 TKN for $90.25, which is not profitable for Bob, so he doesn't liquidate.
5. The prices keeps falling steadily and all this time Alice account is unhealthy but it's not profitable for Bob to liquidate it.
6. Finally, the price stabilizes at $70. Alice assets are now worth $84, while the debt is $95, meaning Alice account is in bad debt. Bob liquidates it partially, swapping 1.2 TKN for 79.8 USDT, meaning Alice account is left with 0 assets and 15.2 USDT debt.

## Impact

Some pools/tokens can be highly volatile, or there might happen to be a period of high volatility, which will cause the difference between the liquidation price (average over `UNISWAP_AVG_WINDOW`) and the current price greater than `LIQUIDATION_INCENTIVE`, making it unprofitable for liquidators to liquidate unhealthy positions. This can cause large bad debt, which is a loss of funds for the other protocol users.

## Code Snippet

Constants being just fixed values, which will be the same for all markets and at all times, which is obviously wrong since different pools/tokens have different volatility and should have different values for these.
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/libraries/constants/Constants.sol#L83-L92

## Tool used

Manual Review

## Recommendation

Consider moving the constants mentioned in this report to a market config rather than hard-coding them in the code. It might also be worth making some of them (like `LIQUIDATION_INCENTIVE`) depend on current volatility:
- `LIQUIDATION_INCENTIVE` - this is the most critical value, which shouldn't be the same across all markets
- `UNISWAP_AVG_WINDOW` - 30 minutes for averages can be too much for some highly volatile markets
- `LIQUIDATION_GRACE_PERIOD` - 2 minutes waiting time before starting liquidation can also be too much for some tokens
- `MAX_LEVERAGE` - max leverage for very volatile tokens should have much lower max leverage, than stable tokens like BTC/ETH, and stablecoins can have higher leverage.
