Dapper Concrete Porcupine

high

# The balance of a courier doesn't get updated when a user burns, leading to an even lower fee effectiveness
## Summary

When a user burns their shares, a portion is sent to their designated courier. However, the courier's balance remains unaltered with `Rewards.updateUserState()`, resulting in reduced fee efficiency. This discrepancy means that when the courier eventually burns, their unclaimable fee is calculated using a formula based on a `high balance * fee rate * time`, instead of recalculating it each time their balance updates, which would be based on the `current balance * fee rate * time`.

## Vulnerability Detail

When users participate in a market's Lender by minting shares through a courier service, they become indebted to that courier for a certain percentage of the shares they've minted. This debt must be settled when users decide to burn their shares.

In addition, users also have the opportunity to claim rewards from the `Factory.sol` protocol. The calculation of these rewards occurs each time a user's balance undergoes changes and is executed by the following line of code: `Rewards.updateUserState(s, a, from, balance);`.

However, there is an important distinction when it comes to couriers. Couriers are not eligible to claim these rewards, and this discrepancy results in less effective fee allocation. Fees are still earmarked for couriers even though they are unable to claim them.

The issue at hand arises due to the fact that fees are not calculated in the same manner for couriers as they are for regular users. Normal users have their fees calculated each time before their balance changes. This approach ensures that they do not accumulate a higher amount of fees over the duration they hold their shares. The code that accomplishes this for regular users is as follows:

```solidity
Rewards.updatePoolState(s, a, newTotalSupply);
Rewards.updateUserState(s, a, from, balance); // @audit they get calculated here <-

uint32 id = uint32(data >> 224);
if (id != 0) {
	uint256 principleAssets = (data >> 112) % Q112;
	uint256 principleShares = principleAssets.mulDivUp(totalSupply_, inventory);

if (balance > principleShares) {
	(address courier, uint16 cut) = FACTORY.couriers(id);

	// ...

	data -= fee;
	balances[courier] += fee;
	emit Transfer(from, courier, fee);
}

```

Conversely, the fees for couriers are not recalculated each time their balance changes due to a user burn. As a result, their unclaimable fees are only calculated when they burn their shares using `withdraw()`. This means that couriers' fees are calculated using a different formula, one that takes into account a high balance after multiple balance additions multiplied by the fee rate and time, as opposed to being recalculated each time their fee balance changes based on their current balance, fee rate, and time.

This vulnerability results in an even higher amount of fees being allocated to stakers, leaving a smaller share of fees for regular users and depleting both regular user rewards and the protocol's funds.

## Impact

This will further diminish the protocol's fee efficiency, resulting in lost funds.

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Lender.sol#L520-L522

## Tool used

Manual Review

## Recommendation

Consider calling `Rewards.updateUserState()` for the courier's account each time a user with a courier burns a portion of their shares.

```solidity
Rewards.updateUserState(s, a, courier, balances[courier]); // @audit Consider adding this <-
balances[courier] += fee;
emit Transfer(from, courier, fee);
```
