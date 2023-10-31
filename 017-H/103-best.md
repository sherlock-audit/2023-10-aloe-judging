Spicy Strawberry Sidewinder

high

# _previewInterest does not properly check for a zero borrowBase leading to major vulnerabilities
## Summary
If `borrowBase` is 0, then `oldBorrows` will also be 0 regardless of the value of `borrowIndex`. The `_previewInterest` function does not check that `borrowBase` is greater than 0 before calculating `oldBorrows`
## Vulnerability Detail
_previewInterest function does not check that borrowBase is greater than 0 before calculating oldBorrows. This could lead to unexpected behavior if borrowBase were 0.
Here is the relevant code:

       function _previewInterest(Cache memory cache) internal view returns (Cache memory, uint256, uint256) {

         // ...

         uint256 oldBorrows = (cache.borrowBase * cache.borrowIndex) / BORROWS_SCALER;

         // ...

       }

If `borrowBase` is 0, then `oldBorrows` will also be 0 regardless of the value of `borrowIndex`. This means no interest will accrue on borrows, even if `borrowIndex` is greater than 1.
This could allow an attacker to manipulate interest accrual on borrows by setting `borrowBase` to 0. For example:
1. Attacker borrows some amount when borrowBase is non-zero. This increments `borrowIndex`.
2. Attacker exploits a vulnerability to set `borrowBase` to 0.
3. Interest continues accruing for lenders, increasing `borrowIndex`. But borrow interest is not accruing since `oldBorrows `stays 0.
4. Attacker can now repay their borrow for less than they should have to.

## Impact
The major impact of not having this check is that the contract could behave unpredictably or incorrectly if `borrowBase` is somehow set to 0.

Specifically:

oldBorrows would be calculated as 0, even if borrowIndex > 0. This would make the utilization rate 0 and could result in incorrect interest accrual.
`newInventory` would equal `lastBalance` instead of `lastBalance + borrows`. This could distort total assets and shares conversions.
The contract would be vulnerable to division by zero if borrowBase stayed 0 while `borrowIndex` increased.
So in summary:

Severity is high because it affects critical accounting logic.


## Code Snippet
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Ledger.sol#L342
## Tool used

Manual Review

## Recommendation
A require check should be added to ensure borrowBase is greater than 0:

       function _previewInterest(Cache memory cache) internal view returns (Cache memory, uint256, uint256) {

         require(cache.borrowBase > 0, "Borrow base cannot be 0");

         // ...

       }

This would prevent borrowBase from being manipulated to 0, ensuring borrow interest accrues properly.
