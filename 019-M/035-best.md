Slow Indigo Woodpecker

medium

# governor can permanently prevent withdrawals in spite of being restricted
## Summary
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
