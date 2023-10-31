Warm Orange Dragon

high

# Bad Debt Attack Can Enitrely Drain Lending Pool When Leverage is Allowed On Uniswap v3 Positions
## Summary

Systems that allow leverage on Uniswap v3 are vulnerable to a bad debt attack which involves creating bad debt in an LP position with corresponding profits in a seperate trading account.

## Vulnerability Detail

The attack is as follows:

- ETH is at $1000 USDC
- In TRADING_ACCOUNT, attacker makes a market order to manipulate price of the ETH-USDC pool to $2001
- In BORROW_ACCOUNT, attacker uses $100k to collateralize a $1M USDC borrow at 90% LTV, and directs that into a single tick liquidity position which is at the both lower and upper tick are around ETH Price of $2000
- TRADING_ACCOUNT sells $1M USDC into the tick of liquidity provided by `BORROW_ACCOUNT`
- When the price returns to the fair price, the BORROW_ACCOUNT has made a $500k loss (it bought $1M of ETH at 2x the fair price), but only $100k of collateral, which is $400k of bad debt
- The TRADING_ACCOUNT has made $500k profit.
- The attacker would not be able to withdraw the $100k of collateral that he deposited into BORROW_ACCOUNT as it now has negative account balance. However he still has $400k of profits minus the cost of manipulating the Uniswap pool.
- 
This type of attack was previously reported as a Critical Bug Bounty to Perpetual Protocol on ImmuneFi, where it was confirmed as valid: https://securitybandit.com/2023/02/07/bad-debt-attack-for-perpetual-protocol/

That report was for a Perpetual Swap Exchange built on top of Uniswap V3, which is not the exact same type of protocol as Aloe. However Aloe's shares the core elements which makes the attack possible:

1. Leverage on top of Uniswap v3 positions
2. No price bands

Without leverage, this attack cannot be profitable, as even with pool manipulation followed by a deposit of liquidity, all trading profits would be cancelled out by corresponding losses in another account. This also holds true when leverage is added, but the losses can be absorbed in a leveraged account which ends up with negative account balance.

Price bands are elaborated on in the "recommendations section" Note that the manipulation price in the example exceeds the "probe prices" which are implemented in Aloe, and Aloe allows liquidity to be added far beyond these prices, which is part of what enables this attack.

Note, the mechanism which detects TWAP Oracle manipulation will not trigger in this case, as the TWAP Oracle does not have to be manipulated in this case, only the Uniswap Price at the current block.

## Impact

Lending pool can be entirely drained by malicious borrower

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Lender.sol#L216-L241

## Tool used

Manual Review

## Recommendation

Add price bands:

Price bands are restrictions that stop trading from happening in leveraged defi protocols when the current trading price differs significantly from oracle price. These are implemented in CLOB perpetual exchanges, for example. After the linked bug was reported to perpetual protocol, they implemented a similar fix to price bands that applies to Uniswap V3. This can be seen in their [limits and caps page in Perpetual Protocol](https://support.perp.com/hc/en-us/articles/5331437456153-Limits-Caps) :

_"You will not be able to add liquidity when the spread between mark and index prices exceeds ±20%. Limit does not affect removing liquidity."_

Since LP's are similar to makers, this fix is analogous to a price band which prevents the adding of maker orders at certain prices which could cause bad debt when the last trading price diverges significantly from the oracle price. In Aloe's case, the price bands should be implemented when the Uniswap v3 `slot0` price (current block) exceeds the TWAP derived probe prices. 

