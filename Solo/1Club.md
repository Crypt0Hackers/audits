# Introduction

A time-boxed security review of the **$1Club** protocol was done by **UniversalCrypto**, with a focus on the security aspects of the application's implementation.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# About **UniversalCrypto**

UniversalCrypto is a team of two independent security researchers [AmaechiEth](https://twitter.com/amaechieth) & [TettehNetworks](https://twitter.com/Tettehnetworks). 

# About **$1Club**

1Club is a digital private members club, utilising ERC721 & ERC20 smart contracts for digital membership access, monthly payment plans & rewards incentives for referals & staking.

The OneClub smart contract is a ERC721 representing the digital membership which uses can mint by providing a 'Voucher' which contains an authorisation signature.

The The1ClubKey smart contract is a ERC20 in a reward token which allows existing members to invite new members and gain referal fees. These keys can also be staked, however for these smart contracts are not in scope for this audit.

The Equity Pool handles funds from staking & accumulated fees within the protocol. After each distribution period has elapsed, stakers are eligibile to rewards based on their stake.

## Observations

The protocol is has numerous priveliged actors, however on initial review of the constructors these addresses will be the deployer of the contract. The protocol achknowledged this and intends to use a multi-sig for every priveliged actor shortly after deployment.

# Threat Model

## Privileged Roles & Actors

Users - Able to trigger fund distribution, mint NFT memberships, create monthly payment plan

Owner - Set critical parameters in `OneClub` smart contract

Pauser - Can pause/unpause certain functions across the protocol

Manager - Can set parameters in `The1ClubPaymentPlans` & `EquityPool` smart contracts

# Severity classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

# Security Assessment Summary

### Scope

The following smart contracts were in scope of the audit:

- `EquityPool`
- `MMP`
- `OneClub`
- `The1ClubKey`
- `The1ClubPaymentPlans`

The following number of issues were found, categorized by their severity:

- Critical & High: 3 issues
- Medium: 4 issues
- Low: 0 issues
- Informational: 10 issues

---

# Findings Summary

| ID     | Title                   | Severity      |
|--------| ----------------------- | ------------  |
| [C-01] | Every payment instalment is double accounted for | Critical      |
| [C-02] | The `payInstalment` function will always revert | Critical      |
| [H-01] | Ether can be locked permanently in the contract     | High          |
| [M-01] | The `payInstalment` function should not have `whenNotPaused` modifier   | Medium        |
| [M-02] | The protocol is heavily centralized upon deployment   | Medium        |
| [M-03] | Unnecessary `receive() external payable {}` can cause lost funds   | Medium        |
| [M-04] | `distributeFunds` will permenently fail once `stakeHolderAddresses` grows sufficiently large   | Medium        |
| [I-01] | Using `SafeMath` when compiler is ^0.8.0| Informational |
| [I-02] | Inconsistent Solidity Pragma| Informational |
| [I-03] | Functions which change critical parameters should emit events| Informational |
| [I-04] | CEI pattern is not followed in `OneClub.sol#safeMint`| Informational |
| [I-05] | Contracts should use interfaces rather than encoding function signatures for external calls for better readability| Informational |
| [I-06] | Use `!=` instead of `<=0` for `uint` comparisons| Informational |
| [I-07] | Incomplete Natspec documentation| Informational |
| [I-08] | Events are missing indexed fields| Informational |
| [I-09] | Referral fees may round to 0 for small `msg.value` amounts| Informational |
| [I-10] | `referralPaid` event will be emitted even if the referral fee is 0| Informational |

# Detailed Findings

# [C-01] Every payment instalment is double accounted for

---

### Severity

**Impact:** High - The user is able to pay half the amount of the monthly instalment and still be considered as having paid the full amount.

**Likelihood:** High - The double accounting of payments will occur on every payment instalment.

### Description

`paymentDetails[msg.sender].cumAmountPaid` tracks the total amount paid by the user. This is incremented by `msg.value` when the user makes a payment instalment. However, `paymentDetails[msg.sender].cumAmountPaid` is also incremented by `paymentDetails[msg.sender].monthlyInstalment` when the user makes a payment instalment. 

When the user is making a payment instalment, `msg.value` is added to `paymentDetails[msg.sender].cumAmountPaid` twice due to incorrect logic.

The first time is here, where the check is made to see if the user's payment is less than the total amount due:
```solidity
if (_paidSoFar < paymentDetails[msg.sender].totalPayment) {
    paymentDetails[msg.sender].lastPaymentTime = block.timestamp;
    paymentDetails[msg.sender].cumAmountPaid += msg.value;
    emit PaymentReceived(msg.sender, msg.value, block.timestamp);
}
```

The second time is here, where the check is made to see if the user's payment is within the payment period:
```solidity
// Check if payment is completed within 6 months payment period
if (
    block.timestamp <= 
    (
        paymentDetails[msg.sender].firstPaymentTime.add( 
        paymentPeriodInSec
    ))
){
    // Update client details
    paymentDetails[msg.sender].lastPaymentTime = block.timestamp;
    paymentDetails[msg.sender].cumAmountPaid += msg.value; 

    // emit PaymentCompleted event
    emit PaymentReceived(msg.sender, msg.value, block.timestamp);
    emit PaymentCompleted(
        msg.sender,
        paymentDetails[msg.sender].cumAmountPaid,
        block.timestamp
    );
}
```

### Recommendations

The logic in this function should be changed to only increment `paymentDetails[msg.sender].cumAmountPaid` by `msg.value` once.

The first check should be used to see if `_paidSoFar` is >= `paymentDetails[msg.sender].totalPayment` and emit `PaymentCompleted` if it is. 

The `PaymentCompleted` event should also be removed from the second check as it may be incorrectly emitted there.

---

# [C-02] The `payInstalment` function will always revert

---

### Severity

**Impact:** High - The user is unable to make a payment instalment.

**Likelihood:** High - The `payInstalment` function will always revert.

### Description

This line in the `payInstalment` function is incorrectly implemented causing it to always revert

```solidity
// Reverts when payment has not been completed within 6 months
revert OneClubPaymentPlans__PaymentNotCompletedWithinPaymentPeriod(); // @audit always reverts?
```

### Recommendations

This line should be part of the previous `if` statement, so that it only reverts when the payment has not been completed within 6 months.

```solidity
// Check if payment is completed within 6 months payment period
if (
    block.timestamp <= 
    (
        paymentDetails[msg.sender].firstPaymentTime.add( 
        paymentPeriodInSec
    ))
){
    // Update client details
    paymentDetails[msg.sender].lastPaymentTime = block.timestamp;
    paymentDetails[msg.sender].cumAmountPaid += msg.value; 

    // emit PaymentCompleted event
    emit PaymentReceived(msg.sender, msg.value, block.timestamp);
    emit PaymentCompleted(
        msg.sender,
        paymentDetails[msg.sender].cumAmountPaid,
        block.timestamp
    );
} else {
    // Reverts when payment has not been completed within 6 months
    revert OneClubPaymentPlans__PaymentNotCompletedWithinPaymentPeriod();
}
```

---

# [H-01] Ether can be locked permanently in the contract

### Severity

**Impact:** High - Ether can become unrecoverable.

**Likelihood:** Medium - This can only occur as a result of an incomplete payment plan.

### Description

When a user doesn't complete their payment plan, the contract will hold their funds.

However, the contract does not have a way to recover these funds.

### Recommendations

Add a way for the contract owner to recover funds from the contract, without putting active payment plan funds at risk. 

This can be done by identifiying expired payment plans, and withdrawing the `paymentDetails[expiredUser].cumAmountPaid` value from the contract.

---

# [M-01] The `payInstalment` function should not have `whenNotPaused` modifier

### Severity

**Impact:** High - The user is unable to make a payment instalment when the contract is paused.

**Likelihood:** Low - This requires interaction from Owner to pause the contract.

### Description

The `payInstalment` function has the `whenNotPaused` modifier which prevents the user from making a payment instalment when the contract is paused. 

If the contract remains paused for a long period of time, the user will default on their payment plan causing them to lose the funds they have already paid.

### Recommendations

Remove the `whenNotPaused` modifier from the `payInstalment` function.

---

# [M-02] The protocol is heavily centralized upon deployment

### Severity

**Impact:** High - It allows priveliged actors to grief or steal funds from users.

**Likelihood:** Low - Priveliged actors must be malicious or compromised

### Description

Upon deployment the following roles are all assigned to `msg.sender` as well as `owner`:

```solidity
_grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
_grantRole(PAUSER_ROLE, msg.sender);
_grantRole(MINTER_ROLE, msg.sender);
_grantRole(MANAGER_ROLE, msg.sender);
```

These roles allow the user to perform the following actions:
- change the treasury address
- pause the contract
- mint tokens
- change the payment period
- change the payment amount
- whitelist users
- set whitelist expiry
- set royalty fees
- set number of instalments per payment plan
- set various contract addresses across the protocol


### Recommendations

Consider the following:
- reducing owner privileges
- assigning roles to different addresses
- using a multi-sig wallet or Governance
- using a timelock on critical functions 

---

# [M-03] Unnecessary `receive() external payable {}` can cause lost funds

### Severity

**Impact:** High - Users can lose funds.

**Likelihood:** Low - This requires users to send funds to the contract without calling a function.

### Description

The `receive() external payable {}` function is unnecessary and can cause users to lose funds if they send funds to the contract without calling a function.

### Recommendations

Remove the `receive() external payable {}` function.

---

# [M-04] `distributeFunds` will permenently fail once `stakeHolderAddresses` grows sufficiently large

### Severity

**Impact:** High - The contract will be unable to distribute funds.

**Likelihood:** Low - This requires a large number of users to be stakeholders

### Description

This unbouded for loop will cause the `distributeFunds` function to fail once `stakeHolderAddresses` grows sufficiently large.

```solidity		
for (uint256 i = 0; i < currentHolderAddresses.length; i++) { 
```

### Recommendations

Consider implementing pull over push payments to avoid this issue, or limit the amount of stakeholders.

---

# [I-01] Using `SafeMath` when compiler is ^0.8.0

Using `SafeMath` in Solidity versions ^0.8.0 as it has built-in underflow / overflow checks.

# [I-02] Inconsistent Solidity Pragma

Different Solidity pragma versions are used throughout the contracts. 

`OneClub` uses `pragma solidity ^0.8.4;`
`MMP` uses `pragma solidity ^0.8.0;`
`EquityPool, The1ClubPaymentPlans & The1ClubKey` use `pragma solidity ^0.8.9;`

Also, locking the pragma helps to ensure that contracts bytecode is compiled the same everytime. Futhermore, an outdated compiler version that might introduce bugs that affect the contract system negatively, consider using the latest version of the Solidity compiler `v0.8.20`

# [I-03] Functions which change critical parameters should emit events

Functions which change critical parameters should emit events to notify users of the change. This is a good practice to ensure that users are aware of the change and can take appropriate action.

# [I-04] CEI pattern is not followed in `OneClub.sol#safeMint`

The CEI pattern is not followed.

Despite there being no apparent re-entrancy risk in this occurence, it is still best security practice to follow the [CEI pattern](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html).

- OneClub.sol#490 `safeMint`

# [I-05] Contracts should use interfaces rather than encoding function signatures for external calls for better readability

There are numerous occassions where the contract encodes function signatures for external calls. This is not a good practice as it makes the code less readable and more prone to errors.

# [I-06] Use `!=` instead of `<=0` for `uint` comparisons

`uint256` type cannot be negative, so it is more gas efficient to use `!=` 

```solidity
if (stakeHolders[_holder].equityTokens <= 0) { 
    revert EquityPool__StakeHolderDoesNotExist();
}
```

# [I-07] Incomplete Natspec documentation

 NatSpec comments improve readability and maintainability of the code. The NatSpec format guidelines can be found [HERE](https://docs.soliditylang.org/en/v0.8.17/natspec-format.html)
 
# [I-08] Events are missing indexed fields

Index event fields make the field more quickly accessible to off-chain tools that parse events. If there are fewer than three fields, all of the fields should be indexed.

# [I-9] Referral fees may round to 0 for small `msg.value` amounts

If the result of the multiplication is small enough the referral fee will round to 0.

```solidity
uint256 referralFee = msg.value.mul(I3_REFERRAL_FEE).div(100); 
```

# [I-10] `referralPaid` event will be emitted even if the referral fee is 0

The following check should include a condition if the referral fee is 0.

```solidity
if (referrals[redeemer] != address(0)) { 
 
...

emit ReferralPaid(referrals[redeemer], redeemer, referralFee);
```