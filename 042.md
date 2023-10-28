Loud Cloud Salamander

medium

# Bad debt liquidation doesn't allow liquidator to receive its ETH bonus (ante)
## Summary

When bad debt liquidation happens, user account will still have borrows, but no assets to repay them. In such case, and when borrow is only in 1 token, `Borrower.liquidate` code will still try to swap the other asset (which account doesn't have) and will revert trying to transfer that asset to callee.

The following code will repay all assets, but since it's bad debt, one of the liabilities will remain non-0:
```solidity
uint256 repayable0 = Math.min(liabilities0, TOKEN0.balanceOf(address(this)));
uint256 repayable1 = Math.min(liabilities1, TOKEN1.balanceOf(address(this)));

// See what remains (similar to "shortfall" in BalanceSheet)
liabilities0 -= repayable0;
liabilities1 -= repayable1;
```

`shouldSwap` will then be set to `true`, because exactly one of liabilities is non-0:
```solidity
bool shouldSwap;
assembly ("memory-safe") {
    // If both are zero or neither is zero, there's nothing more to do
    shouldSwap := xor(gt(liabilities0, 0), gt(liabilities1, 0))
    // Divide by `strain` and check again. This second check can generate false positives in cases
    // where one division (not both) floors to 0, which is why we `and()` with the check above.
    liabilities0 := div(liabilities0, strain)
    liabilities1 := div(liabilities1, strain)
    shouldSwap := and(shouldSwap, xor(gt(liabilities0, 0), gt(liabilities1, 0)))
    // If not swapping, set `incentive1 = 0`
    incentive1 := mul(shouldSwap, incentive1)
}
```
`incentive1` will also have some value (5% from the bad debt amount)

When trying to swap, the execution will revert in TOKEN0 or TOKEN1 `safeTransfer`, because account has 0 of this token:
```solidity
if (liabilities0 > 0) {
    // NOTE: This value is not constrained to `TOKEN1.balanceOf(address(this))`, so liquidators
    // are responsible for setting `strain` such that the transfer doesn't revert. This shouldn't
    // be an issue unless the borrower has already started accruing bad debt.
    uint256 available1 = mulDiv128(liabilities0, priceX128) + incentive1;

    TOKEN1.safeTransfer(address(callee), available1);
    callee.swap1For0(data, available1, liabilities0);

    repayable0 += liabilities0;
} else {
    // NOTE: This value is not constrained to `TOKEN0.balanceOf(address(this))`, so liquidators
    // are responsible for setting `strain` such that the transfer doesn't revert. This shouldn't
    // be an issue unless the borrower has already started accruing bad debt.
    uint256 available0 = Math.mulDiv(liabilities1 + incentive1, Q128, priceX128);

    TOKEN0.safeTransfer(address(callee), available0);
    callee.swap0For1(data, available0, liabilities1);

    repayable1 += liabilities1;
}
```

There are only 2 possible ways to work around this problem:
1. Use very high value of `strain`. This will divide remaining liabilities by large value, making them 0 and will at least repay the remaining assets account has. However, in such case liquidator will get almost no bonus ETH (ante), because it will be divided by `strain`:
```solidity
payable(callee).transfer(address(this).balance / strain);
```

2. Transfer enough assets to bad debt account to cover its bad debt and finish the liquidation successfully, getting the bonus ETH. However, this will still be a loss of funds for the liquidator, because it will have to cover bad debt from its own assets, which is a loss for liquidator.

So the issue described here leads to liquidator not receiving compensation from bad debt liquidations of accounts which have remaining bad debt in only 1 asset. Ante (bonus ETH for liquidator) will be stuck in the liquidated account and nobody will be able to retrieve it without repaying bad debt for the account.

## Vulnerability Detail

More detailed scenario
1. Alice account goes into bad debt for whatever reason. For example, the account has 150 DAI borrowed, but only 100 DAI assets.
2. Bob tries to liquidate Alice account, but his transaction reverts, because remaining DAI liability after repaying 100 DAI assets Alice has, will be 50 DAI bad debt. `liquidate` code will try to call Bob's callee contract to swap 0.03 WETH to 50 DAI sending it 0.03 WETH. However, since Alice account has 0 WETH, the transfer will revert.
3. Bob tries to work around the liquidation problem:
3.1. Bob calls `liquidate` with `strain` set to `type(uint256).max`. Liquidation succeeds, but Bob doesn't receive anything for his liquidation (he receives 0 ETH bonus). Alice's ante is stuck in the contract until Alice bad debt is fully repaid.
3.2. Bob sends 0.03 WETH directly to Alice account and calls `liquidate` normally. It succeeds and Bob gets his bonus for liquidation (0.01 ETH). He has 0.02 ETH net loss from liquidaiton (in addition to gas fees).

In both cases there is no incentive for Bob to liquidate Alice. So it's likely Alice account won't be liquidated and a borrow of 150 will be stuck in Alice account for a long time. Some lender depositors who can't withdraw might still have incentive to liquidate Alice to be able to withdraw from lender, but Alice's ante will still be stuck in the contract.

## Impact

Liquidators are not compensated for bad debt liquidations in some cases. Ante (liquidator bonus) is stuck in the borrower smart contract until bad debt is repaid. There is not enough incentive to liquidate such bad debt accounts, which can lead for these accounts to accumulate even bigger bad debt and lender depositors being unable to withdraw their funds from lender.

## Proof of concept

The scenario above is demonstrated in the test, add it to test/Liquidator.t.sol:
```ts
    function test_badDebtLiquidationAnte() public {

        // malicious user borrows at max leverage + some safety margin
        uint256 margin0 = 1e18;
        uint256 borrows0 = 100e18;

        deal(address(asset0), address(account), margin0);

        bytes memory data = abi.encode(Action.BORROW, borrows0, 0);
        account.modify(this, data, (1 << 32));

        // borrow increased by 50%
        _setInterest(lender0, 15000);

        emit log_named_uint("User borrow:", lender0.borrowBalance(address(account)));
        emit log_named_uint("User assets:", asset0.balanceOf(address(account)));

        // warn account
        account.warn((1 << 32));

        // skip warning time
        skip(LIQUIDATION_GRACE_PERIOD);
        lender0.accrueInterest();

        // liquidation reverts because it requires asset the account doesn't have to swap
        vm.expectRevert();
        account.liquidate(this, bytes(""), 1, (1 << 32));

        // liquidate with max strain to avoid revert when trying to swap assets account doesn't have
        account.liquidate(this, bytes(""), type(uint256).max, (1 << 32));

        emit log_named_uint("Liquidated User borrow:", lender0.borrowBalance(address(account)));
        emit log_named_uint("Liquidated User assets:", asset0.balanceOf(address(account)));
        emit log_named_uint("Liquidated User ante:", address(account).balance);
    }
```

Execution console log:
```solidity
  User borrow:: 150000000000000000000
  User assets:: 101000000000000000000
  Liquidated User borrow:: 49000000162000000001
  Liquidated User assets:: 0
  Liquidated User ante:: 10000000000000001
```

## Code Snippet

`Borrower.liquidate` calculates remaining liabilities after assets are used to repay borrows:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L231-L236

Notice, that if both assets are 0, `liabilities0` or `liabilities1` will still be non-0 if bad debt has happened.

Since either `liabilities0` or `liabilities` are non-0, `shouldSwap` is set to true:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L239-L250

When trying to swap, revert will happen either here:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L263

or here:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L273

## Tool used

Manual Review

## Recommendation

Consider verifying the bad debt situation and not forcing swap which will fail, so that liquidation can repay whatever assets account still has and give liquidator its full bonus.
