Eager Oily Dachshund

medium

# Users can lose rewards if they call claimRewards() before rewardsToken assigned
## Summary

users can lost rewards if they call `claimRewards()` before rewardsToken assigned

## Vulnerability Detail

 `safeTransfer()` in `solmate` library don't check the existence of code at the token address. Because of this if `safeTransfer() `called on a token address that doesn't have a contract in it  will always return success.
rewardsRates and rewardsToken is initially zero. if the protocol intends to reward early users and sets the rewards rate before deploying the actual reward token, users begin accumulating rewards. 
When a user checks their rewards using `lender.rewardsOf(address) `and attempts to claim those rewards using `claimRewards()`, this wont fail. Because of the `safeTransfer()` function which returns true when `rewardsToken` is not set. As a result, users can lose their rewards.

here is the POC:
```solidity
    function test_claimRewardsWithNoToken() public {
        uint56 rate = uint56(bound(100, REWARDS_RATE_MIN, REWARDS_RATE_MAX));
        address holder= address(123456);
        vm.prank(factory.GOVERNOR());
        factory.governRewardsRate(lender, rate);

        Lender[] memory lenders = new Lender[](1);
        lenders[0] = lender;

        // rewardsToken not assigned
        assertEq(address(factory.rewardsToken()), address(0));

        asset.mint(address(lender), 1e18);
        lender.deposit(1e18, holder);

        skip(2 days);
        // Rewards accruing after deposit 
        assertGt(lender.rewardsOf(holder), 0);

        // User can claimRewards without rewardsToken 
        vm.prank(holder);
        factory.claimRewards(lenders, address(this));
        // User lost rewards
        assertEq(lender.rewardsOf(holder), 0);
    }
```

## Impact

User can lost their initial rewards

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/b60c21af24738d517941f18f7caa8c7272f771c5/aloe-ii/core/src/Factory.sol#L242

## Tool used

Manual Review

## Recommendation
Add a check for rewardsToken in `claimRewards()`
```solidity
    function claimRewards(Lender[] calldata lenders, address beneficiary) external returns (uint256 earned) {
        // Couriers cannot claim rewards because the accounting isn't quite correct for them. Specifically, we
        // save gas by omitting a `Rewards.updateUserState` call for the courier in `Lender._burn`
        require(!isCourier[msg.sender]);
+       require(address(rewardsToken)!= address(0));
```