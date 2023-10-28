Spicy Strawberry Sidewinder

high

# Principle accounting is skipped for couriers. This means couriers could manipulate their principle to reduce fees paid to the next courier.
## Summary
Principle accounting is skipped for couriers. This means couriers could manipulate their principle to reduce fees paid to the next courier. 
## Vulnerability Detail
The code does have a vulnerability where couriers can manipulate their principle to reduce fees paid to the next courier.
The relevant code section is in the _burn() function:

       // NOTE: We skip principle update on courier, so if couriers credit 
       // each other, 100% of `fee` is treated as profit and will pass through
       // to the next courier.

This means that when a courier receives a fee, their principle is not updated. So the full amount of the fee is considered "profit" that is subject to the next courier's fee.
Normally, the principle tracks the total assets deposited by a user. This allows calculating the fees correctly - only profit above the principle should be charged.
But by skipping the principle update for couriers, a courier can artificially increase their "profit" and reduce fees paid to the next courier.
For example:
1. Alice deposits 10 tokens. Bob is her courier and takes a 10% fee, so he earns 1 token.
2. Bob's principle should be 1 token, but it is skipped.
3. Bob credits Charlie, who takes a 20% fee.
4. Charlie should earn 0.2 tokens (20% of Bob's profit above principle of 1 token).
5. But since Bob's principle is skipped, Charlie earns 0.2 tokens of the full 1 token (20% of 1 token).

## Impact
The major impact of not tracking principle properly for couriers is that it enables fee manipulation between couriers. Specifically:

Couriers can artificially inflate their principle to reduce fees paid to the next courier in the referral chain. This allows them to unfairly benefit at the expense of other couriers.
Over time, this could significantly reduce the fees collected by downstream couriers. It incentives gaming the system rather than bringing in new business.
If left unchecked, it could undermine the entire courier referral mechanism.
Overall, I would categorize this as a high severity issue. It enables unfair manipulation between couriers and undermines a core mechanism of the protocol. It should be addressed to maintain fairness and incentive alignment.
## Code Snippet
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Lender.sol#L514-L516
## Tool used

Manual Review

## Recommendation
Principles need to be tracked properly for couriers. I suggest this example:

       // Update courier's principle  
       data += fee << 112;

       // Fee calculation uses updated principle
       uint256 profit = balance - (data >> 112) % Q112; 
       uint256 fee = (profit * cut) / 10_000;

This ensures each courier's fees are calculated correctly based on the profit above their principle.

