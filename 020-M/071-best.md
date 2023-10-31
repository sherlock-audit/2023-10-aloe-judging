Bent Orchid Barbel

medium

# Liquidation doesn't consider uniswap fees, so wrong incentive can be calculated
## Summary
Liquidation doesn't consider uniswap fees, so wrong incentive can be calculated
## Vulnerability Detail
When liquidation is called, then first [there is a check that account is not healthy](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L221).

Also, liquidator [incentive is calculated](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L213-L219). It depends [on liabilities](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L216-L217)(amounts that borrower owns to lenders) and [assets inside borrower contract and inside uniswap positions that borrower holds](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L214-L215).

`assets.fixed0` and `assets.fixed1` [is just amount of assets that contract holds at the moment](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L491-L492). While `assets.fluid0C` and `assets.fluid1C` is [amount of token0 and token1 that borrower's uniswap positions hold](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L515-L517).

Pls, note that uniswap LPs also earn fee for the liquidity that they provide. And this fee is not considered in these calculations.
Another point, that we need to note is that when liquidate is called, then [we withdraw everything from uniswap](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L209C63-L209C67), so [fees are withdrawn as well](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L542-L543).

So now we are ready to check how incentive is calculated. This function [just calculates amount that user doesn't have](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/libraries/BalanceSheet.sol#L133-L147) in order to cover all liabilities.

Because `assets.fluid0C` and `assets.fluid1C` doesn't contains fees that means that bigger incentive can be calculated for liquidator. In case if borrower had those positions for a long time(and depending on position sizes), then this fee amount can be big enough to make a loss for borrower.
## Impact
Borrower pays more fees then he should.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
As it's done after liquidity withdraw you can calculate it like this.
```solidity
incentive1 = BalanceSheet.computeLiquidationIncentive(
                TOKEN0.balanceOf(address(this))
                TOKEN1.balanceOf(address(this))
                liabilities0,
                liabilities1,
                priceX128
            );
```