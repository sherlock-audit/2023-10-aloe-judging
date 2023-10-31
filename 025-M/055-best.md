Loud Cloud Salamander

medium

# No handling of L2 sequencer down situation, which can lead to intentional bad debt creation and other malicious actions while sequencer is down or just after it becomes active again
## Summary

The protocol lists L2 networks (Arbitrum, Optimism and Base) as deployment chains. These L2 chains depend on sequencer for most of its functionality. When the sequencer is down for whatever reason, it's still possible to communicate with L2 networks via L1 contracts, but it's very inconvenient and only a few users really use it, meaning that it's unfair to let the protocol keep working as usual when sequencer is down as few users can cause damage to many users during this time (liquidators won't be active etc.). The protocol behavior upon sequencer going back online after a long downtime can also cause all kinds of problems, so it's advised to add grace period when L2 sequencer goes back online

There is functionality to pause some actions if pool manipulation is detected. But there is no functionality to also pause some actions if L2 sequencer is down or when it just came back online.

## Vulnerability Detail

More detailed explanation about the details of L2 networks downtime and how to handle it can be read here:
https://docs.chain.link/data-feeds/l2-sequencer-feeds

The following scenario can happen when L2 sequencer goes down:
1. ETH price = 1000 USDT
2. L2 sequencer goes down for several hours
3. At the time L2 sequencer goes online again, ETH price = 900 USDT
4. Once it comes back up online, uniswap pair average price is still 1000 USDT (and the last uniswap pair price is 1000 USDT a few hours ago, meaning volatility is 0 and average price will slowly go down from 1000 to 900 over 30 minutes)
5. Malicious user can then borrow 1000 USDT and put 1.005 ETH as collateral, effectively selling ETH for 995 USDT, which is way above the current ETH price. Position will be healthy if 1.005 ETH is added to this account as uniswap position with the range [994..1000], it will then be treated as 1005 USDT for health check (since uniswap position will be taken at average price of 1000 instead of current 900).
6. Later, when the average price goes down to 900 USDT, user's account will be liquidated for a bad debt of 100 USDT, causing loss of funds for the other protocol users.

There are also a lot of the other issues possible right after sequencer goes online. For example, it will be unprofitable for liquidators to liquidate accounts if they involve assets swap - liquidator will be forced to buy ETH for 1000 USDT instead of current price of 900 USDT, so liquidations mostly won't work during that time, leaving many unhealthy positions unliquidated.

The same scenario can also be slightly modified to borrow before the sequencer goes offline via direct L1 contract communication. Such user's transaction will be added to queue and will be added to L2 blockchain while sequencer is still offline. There might be even more severe scenarios when user acts via L1 while the sequencer is down.

## Impact

If L2 sequencer goes offline for some time, malicious users can cause all kinds of problems during offline time and in the first minutes after sequencer goes back online (since the price will jump from the last time it is online, but the average price will not catch up until `AVG_WINDOW` time passes). These malicious actions will usually lead to creation of bad debt via different means.

There will also be non-malicious user problems, such as liquidators refusing to liquidate due to unprofitable asset swaps required for liquidaiton.

## Code Snippet

There is a `pause` functionality, but it only allows to pause if pool manipulation is detected:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Factory.sol#L157-L164

But there is no protection of the protocol while L2 sequencer is down or just came back online.

## Tool used

Manual Review

## Recommendation

Consider disabling all borrower actions (liquidate, warn, modify) while sequencer is down. Additionally, disable liquidations and withdrawals in the first AVG_WINDOW seconds after sequencer came back online. Sequencer status and time since last status change can be checked in chainlink examples:
https://docs.chain.link/data-feeds/l2-sequencer-feeds
