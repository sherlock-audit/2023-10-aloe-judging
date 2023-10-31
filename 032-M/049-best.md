Slow Indigo Woodpecker

medium

# Lender.sol: Incorrect rewards accounting for RESERVE address in _transfer function
## Summary
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
