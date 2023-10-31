Bent Orchid Barbel

medium

# Courier can be cheated to avoid fees
## Summary
User can avoid paying fees to courier by providing his address.
## Vulnerability Detail
Courier is the entity that will get some percentage of user's profit, when user withdraws shares. Why someone will be providing fees to that courier? Because courier can provide very comfortable website for user, so he is pleased to pay for that service.

In case if any frontend will deposit on behalf of user, then they should have [at least 1 wei allowance from user](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Lender.sol#L121). In this case they have ability [to set themselves as courier for user](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Lender.sol#L140C83-L140C92).

Courier is set inside `_mint` function. There is one important thing: in case if user already has shares, [then courier will not be set](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Lender.sol#L453-L456), which means that whoever did that deposit, he will not receive any fees in the end.

So how user can use this? The most simple way is to frontrun deposit call and transfer some amount of shares to his account from another account. As result, his shares amount will not be 0 anymore and fees will be avoid.

Another problem is that when user has used one website and then switched to another one, then new service expects that user will pay fees for using it, but in reality after  new deposit, only old frontend will receive all fees.
## Impact
User can cheat couriers.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
I guess that deposit function should check if provided courier will be set or not and early revert if not.