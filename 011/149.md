Trendy Strawberry Skunk

medium

# `Lender.sol` is not fully compliant with `EIP2612`
## Summary
`Lender.sol` is not fully compliant with `EIP2612`

## Vulnerability Detail
Per the contest readme.md, `Lender.sol` is compliant with `EIP2612`.

> Is the code/contract expected to comply with any EIPs? Are there specific assumptions around adhering to those EIPs that Watsons should be aware of?
The Lender complies with ERC4626 and EIP2612.

As per the [`ERC-2612`](https://eips.ethereum.org/EIPS/eip-2612) which is used as permit Extension for EIP-20 Signed Approvals.

> Specification
Compliant contracts must implement 3 new functions in addition to EIP-20:
function permit(address owner, address spender, uint value, uint deadline, uint8 v, bytes32 r, bytes32 s) external
function nonces(address owner) external view returns (uint)
function DOMAIN_SEPARATOR() external view returns (bytes32)

1) `Permit()` is used in contract and the function is incompliance with EIP2612.
2) `nonces()` is missing therefore the `Lender.sol` is not in compliance with EIP2612. It should be noted here that `nonce` function is `MUST` requirement of `EIP2612`. Therefore this requirement can not be omitted. Per the defination of `MUST` as per [RFC-2119](https://datatracker.ietf.org/doc/html/rfc2119)

> MUST   This word, or the terms "REQUIRED" or "SHALL", mean that the definition is an absolute requirement of the specification.

Therefore, nonce() must be added in contract for proper compliance.

3) `DOMAIN_SEPARATOR()` is used incorrectly in contract. It should be noted that `Lender.sol` inherits `Ledger.sol` and `DOMAIN_SEPARATOR()` is a part of `Ledger.sol`

`DOMAIN_SEPARATOR()` used in the contract is given as below,

```Solidity
    function DOMAIN_SEPARATOR() public view returns (bytes32) {
        return
            keccak256(
                abi.encode(
                    keccak256("EIP712Domain(string version,uint256 chainId,address verifyingContract)"),
                    keccak256("1"),
                    block.chainid,
                    address(this)
                )
            );
    }
```

The above given function does not comply the `EIP2612` as the `EIP2612` states,

> DOMAIN_SEPARATOR is defined according to EIP-712. The DOMAIN_SEPARATOR should be unique to the contract and chain to prevent replay attacks from other domains, and satisfy the requirements of EIP-712.

The `DOMAIN_SEPARATOR` should be like this,

```Solidity
DOMAIN_SEPARATOR = keccak256(
    abi.encode(
        keccak256('EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)'),
        keccak256(bytes(name)),
        keccak256(bytes(version)),
        chainid,
        address(this)
));
```

Therefore, `DOMAIN_SEPARATOR()` should be corrected.

## Impact
`Lender.sol` breaks the design integration with `EIP2612` and could result in expected behaviour which is not desired. Being EIP2612 is the core requirement of `Lender.sol`

## Code Snippet
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/Ledger.sol#L128

## Tool used
Manual Review

## Recommendation
Correctly follow EIP2612. Add the `nonces()` function which is a must requirement here also correct the `DOMAIN_SEPARATOR()` function as stated in EIP2612