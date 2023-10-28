Blunt Sky Gibbon

medium

# Transferring an insufficient amount on `Lender.deposit` causes a `.transferFrom` of the full amount, without minting extra shares
## Summary
Transferring an insufficient amount on `Lender.deposit` causes a `.transferFrom`
of the full amount, without minting extra shares

## Vulnerability Detail

consider this test, added to `test/Lender.t.sol`

```solidity
function test_deposit_with_transferFrom() public returns (address alice) {
    alice = makeAddr("alice");
    deal(address(asset), alice, 10000);
    hoax(alice, 1e18);
    asset.transfer(address(lender), 90);
    hoax(alice);
    asset.approve(address(lender), 10000);
    hoax(alice);
    uint256 shares = lender.deposit(100, alice);
    assertEq(shares, 100);
    assertEq(lender.totalSupply(), 100);
    assertEq(asset.balanceOf(alice), 9900);
}
```

the last assert would fail, since the protocol is minting 100 shares but
received the first 90 of `asset` in the transfer, plus 100 more with the
`transferFrom`, for a total reduction in alice's balance of 190.

Furthermore, shares are only minted for the initially specified amount of 100,
and this value could be captured by anyone (ie, if bob calls `deposit` with an
amount of 90, shares will be awarded to him at no cost)

## Impact

It's worth noting that this is not a vector for sandwiching/front running since
the amount of `asset` to deposit into the contract, and not the amount of shares
expected in return, is what's passed as a parameter and determines the
`transferFrom` amount. Therefore it's not possible to cause another account's
`deposit` tx to have an insufficient transfer by executing an interest
accrual before it.

I'm still submitting this as a Medium because funds could definetely (but
unlikely) be griefed from users, in a way that's easily mitigable by a change in
the protocol.

## Code Snippet

[from  `src/Lender.sol`](https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Lender.sol#L151) line 149 onwards:
```solidity
bool didPrepay = cache.lastBalance <= asset_.balanceOf(address(this));
if (!didPrepay) {
    asset_.safeTransferFrom(msg.sender, address(this), amount);
}
```

## Tool used

Manual Review

## Recommendation

Either remove the possibility of not doing a full prepay and revert the
transaction, or perform the `transferFrom` for only
`cache.lastBalance - asset_.balanceOf(address(this)) ` instead of the full
`amount`
