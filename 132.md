Dapper Concrete Porcupine

medium

# The protocol doesn't maximise it's pool balances when executing operations, leading to them favouring the user
## Summary

Certain balances need to be rounded differently to prevent users from benefiting at the expense of the protocol.

## Vulnerability Detail

The protocol contains several instances where balance calculations have a direct impact on users. The problem lies in the rounding methodology, which should favour the protocol rather than the user.

For instance, consider the `_previewInterest()` function, which does not maximise the values of `newInventory` and `newTotalSupply`.

```solidity
uint256 newInventory = cache.lastBalance + (cache.borrowBase * cache.borrowIndex) / BORROWS_SCALER;
uint256 newTotalSupply = Math.mulDiv(
    cache.totalSupply,
    newInventory,
    newInventory - (newInventory - oldInventory) / rf
);

```

In the `borrow()` function, it's essential to round up the `units` added to a user's balance to prevent precision loss that favours the protocol over users.

```solidity
units = (amount * BORROWS_SCALER) / cache.borrowIndex;

```

## Impact

These discrepancies in the protocol lead to users receiving slightly more value than intended, which can accumulate over time as they will happen on almost every type of interaction.

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Ledger.sol#L359-L364

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Lender.sol#L225

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Lender.sol#L526

## Tool used

Manual Review

## Recommendation

Consider introducing a round-up mechanism in the `_previewInterest()` function, similar to the approaches in `_convertToShares` and `_convertToAssets`. This will ensure that the appropriate rounding method is applied in various cases, aligning with the user's interests.

Additionally, consider modifying the `units` calculations in the `borrow()` function as follows:

```solidity
// @audit Proper rounding for the use case:
units = amount.mulDivUp(BORROWS_SCALER, cache.borrowIndex);

```

These adjustments will address the balance rounding issue and uphold the user's intended interests.