Tricky Heather Swallow

medium

# Users may lose rewards accrued before enrolling as courier
## Summary

Users who had rewards before enrolling as a courier will lose those rewards because they are not claimed before setting the user as a courier.

## Vulnerability Detail

After enrolling as a courier, a user will no longer be able to call the `claimRewards` function in `Factory.sol` due to the function reverting if the sender is a courier.

```solidity
require(!isCourier[msg.sender]);
```

The docs on [line 247](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Factory.sol#L247) states that the user will not be eligible for rewards after enrolling. Users may expect that the rewards accrued up to the point of enrollment are still claimable and unknowingly lock themselves out of claiming their past rewards.

## Impact

A user may lose rewards accumulated up to the point of becoming a courier if they do not claim beforehand. Since the `claimRewards` function on `Lender.sol` may only be called by the factory, the user will have no way to claim their previous rewards.

## Code Snippet

[Factory.sol#L254-L266](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Factory.sol#L254-L266)

[Factory.sol#L228-L243](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Factory.sol#L228-L243)

## Tool used

Manual Review

## Recommendation

Consider adding claiming functionality to the `enrollCourier` function that would allow the user to claim from an array of lenders supplied by the user.