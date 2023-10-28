Loud Cloud Salamander

high

# Large bad debt can cause bank run since there is no loss socialization mechanism
## Summary

When a large bad debt happens in the system, it is "stuck" in the system forever with no incentive to cover it. User, whose account goes into bad debt has no incentive to add funds to it, he will simply use a new account. And the other users also don't have any incentive to repay bad debt for such user.

This means that the other users will never be unable to withdraw all funds due to this bad debt. This can cause a bank run, since the first users will be able to withdraw, but the last users to withdraw will be unable to do so (will lose funds), because protocol won't have enough funds to return them since this bad debt will remain unreturned infinitively and will, in fact, keep accumulating even more bad debt.

## Vulnerability Detail

If some users takes a huge borrow and later there is a quick price drop, which will cause user's account to fall into a large bad debt, there will be no incentive for the liquidators to fully liquidate user, because the assets he has won't be enough to compensate the liquidator, meaning partial liquidations will bring user to a state with 0 assets but still with borrowed assets (bad debt).

Since there is no incentive from any users to repay these assets, this borrow will remain in the system forever, meaning this is basically a loss of funds for the other users. If this accumulated bad debt is large enough, the users will notice this and might start a bank run, because the users who withdraw first will be able to do so, but those who try to withdraw later will be unable to do so, because the protocol won't have funds, only the "borrowed" amounts which will never be returned due to bad debt (those accounts only having borrow/debt without any assets).

## Impact

Bad debt accumulation can lead to a bank run from the users with the last users to withdraw losing all their funds without any ability to recover it.

## Code Snippet

`Borrower.liquidate` doesn't care about bad debt, leaving it up to liquidators to choose correct `strain` amount to only get all assets the user has after liquidation, so that it doesn't revert:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L257-L277

## Tool used

Manual Review

## Recommendation

Consider introducing bad debt socialization mechanism like the other lending platforms (bad debt will then reduce the `borrowIndex`, thus socializing the loss between all lenders). It will also help clear borrow balance from bad debt accounts, preventing it to further accumulate even more bad debt.