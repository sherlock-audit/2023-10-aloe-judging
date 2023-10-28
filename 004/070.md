Bent Orchid Barbel

medium

# Borrower can dos liquidations
## Summary
Borrower can dos liquidations by resetting warn status every time 
## Vulnerability Detail
In order to liquidate borrower, liquidator should call `warn` function first, which [will set time](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L171) until borrower has ability to adjust posiiton. Only [after that time](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L254) liquidator can call `liquidate` and get bonus for liquidation.

But there is one more thing: in case if swap is not needed during liquidation, [then warn check is skipped](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L252).

Borrower can use such thing, together with `strain` param in order to reset `warn`.
`strain` param is used to adjust amount that liquidator wants to swap. So [liabilities are divided by this param](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L245-L246). And later xor is used to determine if swap is needed, in case if both liabilities are 0, then no swap is needed. So borrower should provide such a `strain` params, that is bigger than `max(liabilities0, liabilities1`), so dividing will give 0 for both cases and swap will be skipped.

As result user's uniswap positions is not waithdrawn and liquidators should call `warn` again to start next 2 minute warning. On chains with cheap gas price, borrower can do this as many time as he wishes.
## Impact
User can avoid being liquidated even if his position is unhealthy.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
You can validate `strain` param, to be smth not big(as you expect that small numbers will be used). Or do not allow to call liquidate without `warn` period passed when there will be no swap.