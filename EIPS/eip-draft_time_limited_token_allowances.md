---
eip: eip-draft_time_limited_token_allowances
title: time-limited token allowances 
description: Extend EIP-20 approve() functionality in a way that the user can set a block number limit after which the approval expires.
author: Panagiotis Vryonis (@vrypan)
discussions-to: https://ethereum-magicians.org/t/time-limited-erc-20-approvals/7749
status: Draft
type: Standards
category (*only required for Standards Track): ERC
created: 2021-12-14
requires (*optional): 20, 165
---

## Abstract
The following standard extends EIP-20 in a way that approvals are valid for a specific number of blocks. The standard is designed in a way that is backwards compatible with EIP-20.

## Motivation
In order to maintain a secure and healthy wallet, users must regularly check the token allowances they have authorised and revoke them if needed.

This task is not trivial for most users. It requires investment in time. For most users, this means using a third party service which may introduce additional security risks. Last, but not least, each revocation costs in gas fees.

We should also take into consideration that a user's decision to approve an allowance is an act of trust based on factors that may change in the future, such as who controls the ``_spender`` address, or the value of the ERC token. 

This EIP assumes that many of these risks will be reduced if the allowance expires after a period of time.

## Specification
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

This EIP extends EIP-20. All statements of EIP-20 are valid, unless specified otherwise.

### New functionality
A contract that implements this EIP must internally define two constants:
- DEFAULT_ALLOWANCE_BLOCKS
- DEFAULT_EXPIRATION_BLOCK

Two new functions are added to the ERC20 ABI:

#### approveWithExpiration
```sol
function approveWithExpiration(address _spender, uint256 _value, uint256 _blocks) public returns (bool success)
```

Just like ERC20 ``approve`` this function allows `_spender` to withdraw from your account multiple times, up to the `_value` amount.

When called, the contract sets internally ``expiration_block = block.number + _blocks``. 

Spending MUST throw if ``block.number > expiration_block``.

Note: 0 is a valid value for ``_blocks``. It approves spending that happens only in the same block as the approval, which may be useful in cases such as flash loans. 

#### allowanceWithExpiration
```sol
function allowanceWithExpiration(address _owner, address _spender) public view returns (uint256 remaining, uint256 expiration_block)
```
Just like ERC20 `allowance`, this function eturns the amount which `_spender` is still allowed to withdraw from `_owner`. **In addition**, it returns the max block number that the allowance is valid for.

### Additional specifications for ERC20 functions 

The following functions keep the same signature as in ERC20, but their behaviour is re-defined to support allowance expiration. 

All changes are designed to be backwards compatible.

#### approve
```sol
function approve(address _spender, uint256 _value) public returns (bool success)
```
Allows `_spender` to withdraw from your account multiple times, up to the `_value` amount. If this function is called again it overwrites the current allowance with `_value`.

**New functionality:** ``approve`` MUST set the internal value for ``expiration_block`` to ``block.number + DEFAULT_ALLOWANCE_BLOCKS``.

#### allowance 
```sol
function allowance(address _owner, address _spender) public view returns (uint256 remaining)
```

Returns the amount which `_spender` is still allowed to withdraw from `_owner`. 

**New functionality:** 
- if `expiration_block` is not defined, ``DEFAULT_EXPIRATION_BLOCK`` is assumed. This allows ERC20 contracts already holding allowances to be upgraded without breaking them.
- if `block.number` is greater than `expiration_block`, this function returns ``0``.

#### transferFrom
```sol
function transferFrom(address _from, address _to, uint256 _value) public returns (bool success)
```
Transfers `_value` amount of tokens from address `_from` to address `_to`, and MUST fire the `Transfer` event. 

**Added functionality**: 
- if `expiration_block` is not defined, ``DEFAULT_EXPIRATION_BLOCK`` is assumed. This allows ERC20 contracts already holding allowances to be upgraded without breaking them.
- if `block.number` is greater than `expiration_block`, this function throws.

## Rationale

Allowance expiration is based on block numbers and not timestamps in order to allow uses where ``approveWithExpiration`` and ``transferFrom`` are executed in a single transaction (for example, flash loans). In this case, ``approveWithExpiration`` can be called with ``_blocks=0``.

In order to maintain backwards compatibility with ERC20, each smart contract must define ``DEFAULT_ALLOWANCE_BLOCKS`` (the default value for ``_blocks`` when ``approve`` is used). An alternative would be to make the default value of ``uint(-1)`` part of this EIP. However, this would limit smart contracts that want to provide a default value that better suits their and their users needs.

Contracts are also free to define ``DEFAULT_EXPIRATION_BLOCK`` based on their needs. While the obvious choice may be to set this to uint(-1), developers may set it to other values. For example, when upgrading an existing ERC-20 contract, ``DEFAULT_EXPIRATION_BLOCK`` may be set to contract deployment block.number + X in order to give a grace period of X blocks until users have to approve a new allowance, or set it to 0 to cancel all existing allowances, and force users to approve new ones.

## Backwards Compatibility
This EIP was designed to be backwards compatible with EIP-20: Any contract that implements it can be used as an EIP-20 contract, without any modification by the API consumer.

## Security Considerations
This EIP does not introduce additional security considerations compared to EIP-20.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
