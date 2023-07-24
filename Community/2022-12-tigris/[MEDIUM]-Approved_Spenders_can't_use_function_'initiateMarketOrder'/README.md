# Original link
https://github.com/code-423n4/2022-12-tigris-findings/issues/280
# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L163


# Vulnerability details

## Impact

`EIP-2612` allows a `signer` to create a signature which specifies any address as `spender`, which should allow the `spender` to use the signature created to validate transactions. However, in `Trading.sol` this `spender` cannot use `initiateMarketOrder` preventing them from opening positions on behalf of the `signer`. 

## Proof of Concept

Originally in `07.Trading.js` test file, a signature is created like this which passes:
```
permitSig = await signERC2612Permit(owner, MockDAI.address, owner.address, Trading.address, ethers.constants.MaxUint256);
```

For this POC we will assume the `owner` wants to approve `user` as a spender:
```
permitSig = await signERC2612Permit(owner, MockDAI.address, user.address, Trading.address, ethers.constants.MaxUint256);
```

This should allow `user` to open positions using tokens from `owner`.

On Line 185: https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/test/07.Trading.js#L185

The `owner` opens a trade using `owner.address` as the trader.

In this example the `spender` should be able to open a trade using the signature created by the `owner`, however, in all cases, this will revert.
```
await trading.connect(user).initiateMarketOrder(TradeInfo, PriceData, sig, PermitData, owner.address);
```
```
await trading.connect(user).initiateMarketOrder(TradeInfo, PriceData, sig, PermitData, user.address);
```
```
await trading.connect(owner).initiateMarketOrder(TradeInfo, PriceData, sig, PermitData, user.address);
```
## Tools Used

Manual Audit

## Recommended Mitigation Steps

Specify whether signatures with a different `signer` to `spender` will not be validated.