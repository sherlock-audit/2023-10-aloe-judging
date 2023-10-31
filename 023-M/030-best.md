Loud Cloud Salamander

medium

# Liquidator with high `strain` will liquidate the account fully if no assets swap is required, but will only get the bonus equal to `1/strain` ETH
## Summary

`liquidate` function has `strain` parameter, which specifies the ratio (`1/strain`) of the position to be liquidated. This parameter is only used if liquidator has to swap some assets. However, if removing uniswap liquidity and repaying the debt is enough (no swap required from liquidator), liquidator will still get the `1/strain` bonus ETH even though he has fully finished the account liquidation.

## Vulnerability Detail

`Borrower.liquidate` sends the ETH bonus to liquidator divided by strain regardless of whether it was used or not:
```solidity
payable(callee).transfer(address(this).balance / strain);
```

## Impact

If the liquidation has finished completely (account is healthy again), liquidator will get smaller amount than he should (he should get full bonus since he has completed liquidation - regardless of strain parameter, but he will get bonus / strain instead).

## Code Snippet

`Borrower.liquidate` send ETH bonus to liquidator divided by strain:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L283

## Tool used

Manual Review

## Recommendation

Consider verifying if account is healthy after liquidation and send full ETH bonus to liquidator if it's healthy (which is logical, since the liquidator has acomplished its task fully).