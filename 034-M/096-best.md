Refined Alabaster Owl

medium

# Couriers may transfer lender tokens to another address and earn rewards
## Summary
By design, couriers are not allowed to earn rewards because the accounting isn't quite correct for them. 
```solidity
// NOTE: We skip rewards update on the courier. This means accounting isn't
// accurate for them, so they *should not* be allowed to claim rewards. This
// slightly reduces the effective overall rewards rate.
```
However, there is nothing restricting a courier from transferring their lender tokens to another address (that is not a courier) to continue earning rewards. 

## Vulnerability Detail
`_transfer` prevents any transfer between lenders with a courier.
```solidity
/// Lender.sol
/// @dev Transfers `shares` from `from` to `to`, iff neither of them have a courier
    function _transfer(address from, address to, uint256 shares) private {
}
```
However, there are no measures to prevent the couriers from transferring out their lender tokens to another non-courier address. This non-courier address can then bypass the check for couriers in `claimRewards()` and receive rewards.

```solidity
/// Factory.sol
function claimRewards() {
  require(!isCourier[msg.sender]);
```
## Impact
The allocated amount of reward tokens will be depleted faster as a result of couriers also claiming the reward, reducing the net received rewards for legitimate claimers.

## Code Snippet
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Factory.sol#L231
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Factory.sol#L231

## POC
Add this test to LenderReferrals.t.sol
https://gist.github.com/chewonithard/3693ecad98ebed9d839e87c1744b64cc

## Tool used

Manual Review

## Recommendation
Similar to how lenders with couriers are prevented from doing `_transfer`, also prevent couriers from doing `_transfer` by adding:
`require(!isCourier[msg.sender]);`
