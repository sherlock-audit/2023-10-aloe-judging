Dapper Concrete Porcupine

high

# The whole ante balance of a user with a very small loan, who is up for liquidation can be stolen without repaying the debt
## Summary

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
