Warm Orange Dragon

medium

# Liquidations Allowed When Paused
## Summary

When the protocol is paused, liquidations are still allowed while reducing modifying positions is disallowed, leading to unfair liquidations.

## Vulnerability Detail

When the protocol is paused, the modify function will revert due to this line:

```solidity
            require(
                //@question was is ante?
                seemsLegit && (block.timestamp > pausedUntilTime) && (address(this).balance >= ante),
                "Aloe: missing ante / sus price"
            );
```

This means that users cannot modify their positions and increase their collateral to avoid liquidations. However, the `warn` and `liquidate` functions do not have the same check which measn that liquidations are still allowed when the protocol is paused. This results in unjust liquidations.

Similar to this issue from Blueberry Contest: [Jeiwan - Liquidations are enabled when repayments are disabled, causing borrowers to lose funds without a chance to repay](https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/290)



## Impact

Unfair liquidations as users cannot modify their position while liquidations are still enabled.

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L299-L327

## Tool used

Manual Review

## Recommendation

Implement a pause check on both the `liquidate` and `warn` functions
