# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/107
# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/LPDA.sol#L58-L59
https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/FixedPrice.sol#L57-L74
https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/OpenEdition.sol#L57-L72


# Vulnerability details

## Impact

The variable `_amount` is `uint256` but the variable `amount` is `uint24` so the variable `amount` the full value may not be stored.

These calculations and operations may produce incorrect results if the `amount` variable has a different value than what was intended.

This could still cause the contract to behave unexpectedly and potentially lead to financial losses for users of the contract or incorrect balances being stored.


## Proof of Concept

This possibility exists because the amount variable is defined as an `uint48` type, which has a narrower range of possible values than the `_amount` parameter's `uint256` type. 

If the value assigned to the `amount` variable is greater than the maximum value that can be stored in an `uint48` type, the value will be truncated. This truncation may result in the amount variable having a different value than intended.

## Tools Used

Manual Audit

## Recommended Mitigation Steps

```
if (_amount <= uint48.max()) {
  uint48 amount = uint48(_amount);
  ...
} else {
  // Handle _amount being too large
}
```