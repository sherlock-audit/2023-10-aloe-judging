Loud Cloud Salamander

medium

# `Borrower.modify` callee callback returned positions value of 0 is treated as "no change", making it impossible to return "no positions" value
## Summary

`Borrower.modify` callee callback returns the zipped positions value, which holds an array of 3 uniswap positions. The return value of "0" is special and treated as "no change". However, this special value makes it impossible to indicate all positions are withdrawn, because the zipped value of "no positions" (positions array of [0,0,0,0,0,0]) will return 0, which will be treated as "no change" instead of "no positions".

## Vulnerability Detail

When `modify` calls callee, slot0 positions value doesn't change if returned value = 0:
```solidity
uint208 positions = callee.callback(data, msg.sender, uint208(slot0_));
assembly ("memory-safe") {
    // Equivalent to `if (positions > 0) slot0_ = positions`
    slot0_ := or(positions, mul(slot0_, iszero(positions)))
}
```

However, if user had some positions before `modify`, but decided to withdraw them and return 0 (meaning there are no uniswap positions by the user), this will be treated as "no change" instead of "no positions", thus slot0 will keep previous positions value.

While this doesn't affect anything if the user has actually withdrawn uniswap position (all calculations with these unexisting positions will simply return 0 liquidity and be ignored), however the positions value stored in slot0 doesn't mean these are all user positions - it simply means these are positions user wants to be considered as collateral (and which can be seized during liquidation). If user wants to remove all uniswap positions from collateral while still keeping all of them, there is a valid case possible which will cause loss of funds for the user:

1. User deposits uniswap position (`modify` returns [100, 200, 0, 0, 0, 0] zipped uniswap positions array)
2. User decides to "withdraw" this position from collateral while still keeping the uniswap position itself. So a new `modify` doesn't do anything, but returns now empty positions array ([0,0,0,0,0,0]). User expects that his positions are now not part of the collateral and will not be seized during liquidation.
3. Later the user is liquidated and his [100,200] positions is seized, because in step 2 the `callee.callback` returned 0, which means that user positions array didn't change and remained at [100,200,0,0,0,0].
4. As a result, user has unexpectedly lost his uniswap position which should not be part of the collateral.

## Impact

In an edge case, when user wants to remove all uniswap positions from collateral without actually removing them from uniswap itself, this operation will be ignored and user will unexpectedly lose all these uniswap positions he didn't think were part of the collateral.

## Code Snippet

`Borrower.modify` doesn't change slot0 if returned positions value is 0 (but "no positions" array of [0,0,0,0,0,0] also returns 0 value and is ignored instead of changing slot0 to empty positions array)
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L306-L310

## Tool used

Manual Review

## Recommendation

Consider using a different magic number for "no change", or maybe introduce some other variable (like "no change" bit) as part of the unused positions bits.