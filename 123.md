Dapper Concrete Porcupine

medium

# The functionality of payable(address).transfer will be compromised if the cost of SLOAD increases
## Summary

Any increase in the cost of the SLOAD opcode will cause native coin transfers with `payable.transfer()` to run out of gas and fail.

## Vulnerability Detail

The `payable.transfer()` function forwards a fixed amount of 2300 gas. The gas cost of EVM instructions may change significantly during hard forks which may break already deployed contract systems that make fixed assumptions about gas costs. For example, EIP 1884 broke several existing smart contracts due to a cost increase of the SLOAD opcode from 200 to 800 gas units.

Read more about EIP-1884 here:

https://eips.ethereum.org/EIPS/eip-1884

## Impact

The increase in the cost of the SLOAD opcode will render all `payable.transfer()` calls non-functional.

## Code Snippet

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L283

https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Borrower.sol#L434

## Tool used

Manual Review

## Recommendation

Consider changing all occurrences of `payable.transfer()` with `address.call()` with value in order to mitigate the issue.

```solidity
recipient.call{ value: address(this).balance }();
```