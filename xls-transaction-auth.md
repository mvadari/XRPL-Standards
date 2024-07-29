<pre>
Title:       <b>Account Permission Delegation</b>
Revision:    <b>1</b> (2024-07-29)

Author:      <a href="mailto:mvadari@ripple.com">Mayukha Vadari</a>

Affiliation: <a href="https://ripple.com">Ripple</a>
</pre>

# Account Permission Delegation

## Abstract

This proposal introduces a delegated authorization mechanism to enhance the flexibility and usability of XRPL accounts. Currently, critical issuer actions, such as authorizing trustlines, require direct control by the account holder, limiting operational efficiency and hindering complex use cases. By enabling account holders to selectively delegate specific permissions to other accounts, this proposal aims to empower users without compromising account security. This mechanism will unlock new possibilities for XRPL applications, such as multi-party workflows and advanced account management strategies.

## 1. Overview

We propose:
* Creating a `TransactionAuth` ledger object.
* Creating a `TransactionAuth` transaction type.

We also propose modifying the transaction common fields.

This feature will require an amendment, tentatively titled `featureTransactionAuth`.

### 1.1. Terminology

* **Permissions** - all the permissions you can have, could be a transaction type or something more granular

### 1.2. Permissions

The different types of permissions that can be supported are explained [in XLS-otherd](https://gist.github.com/mvadari/a8d76f0c4e3aa54eb765f08bcacc5316).

## 2. On-Ledger Object: `TransactionAuth`

This object represents a set of permissions that an account has delegated to another account, and is modeled to be similar to [`DepositPreauth` objects](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/depositpreauth).

### 2.1. Fields

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`LedgerIndex`| ✔️|`string`|`Hash256`|The unique ID of the ledger object.|
|`LedgerEntryType`| ✔️|`string`|`UInt16`|The ledger object's type (`TransactionAuth`)|
|`Account`| ✔️|`string`|`AccountID`|The account that wants to authorize another account.|
|`Authorize`| ✔️|`string`|`AccountID`|The authorized account.|
|`Permissions`| ✔️|`string`|`STArray`|The transaction permissions that the account has access to.|

#### 2.1.1. Object ID

The ID of this object will be a hash of the `Account` and `Authorize` fields, combined with the unique space key for `TransactionAuth` objects, which will be defined during implementation.

#### 2.1.2. `Permissions`

Could be a transaction type value, or a more specific value. See other XLS spec about granular permissions.

The array will have a maximum length of 10.

## 3. Transaction: `TransactionAuth`

This object represents a set of permissions that an account has delegated to another account, and is modeled to be similar to the [`DepositPreauth` transaction type](https://xrpl.org/docs/references/protocol/transactions/types/depositpreauth).

### 3.1. Fields

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`TransactionType`| ✔️|`string`|`UInt16`|The transaction type (`TransactionAuth`).|
|`Account`| ✔️|`string`|`AccountID`|The account that wants to authorize another account.|
|`Authorize`| |`string`|`AccountID`|The authorized account.|
|`Permissions`| ✔️|`string`|`STArray`|The transaction permissions that the account has been granted.|

### 3.1.1. `Permissions`

This transaction works slightly differently from the `DepositPreauth` transaction type. Instead of using an `Unauthorize` field

### 3.2. Failure Conditions

* `Permissions` is too long
* Any of the specified permissions are 
* Both or neither `Authorize` and `Unauthorize` are specified

### 3.3. State Changes

## 4. Transactions: Common Fields

### 4.1. Fields

As a reference, [here](https://xrpl.org/docs/references/protocol/transactions/common-fields/) are the fields that all transactions currently have.

<!--There are too many and I didn't want to list them all, it cluttered up the spec - but maybe it can be a collapsed section?-->

We propose these modifications:

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`OnBehalfOf`| |`string`|`AccountID`|The account that the transaction is being sent on behalf of.|

### 4.2. Failure Conditions

* The `OnBehalfOf` account doesn't exist.
* The `OnBehalfOf` account hasn't authorized the transaction's `Account` to send transactions on behalf of it.
* The `OnBehalfOf` account hasn't authorized the transaction's `Account` to send this particular transaction type/granular permission on behalf of it.

### 4.3. State Changes

The transaction succeeds as if the transaction was sent by the `OnBehalfOf` account. 

## 5. Examples

## 6. Invariants

* An account should never be able to send a transaction on behalf of another account without a valid `TransactionAuth` object.

## 7. Security

Delegating permissions to other accounts requires a high degree of trust, especially when the delegated account can potentially access funds or charge reserves.

On the other hand, this mechanism also offers a granular approach to authorization, allowing accounts to selectively grant specific permissions without compromising overall account control. This approach provides a balance between security and usability, empowering account holders to manage their assets and interactions more effectively.

## n+1. Open Questions

* Does there need to be a new account flag for this?
	* I don't think so - signer lists don't need a new account flag
* Could this be abused by scammers and other malicious people, who could convince unsuspecting users to give other accounts permissions to drain the account?
	* This may need to have heavy warnings associated with it, especially on the `Payment` transaction
	* I guess signer lists (and especially multiple signer lists) have the same possible problem?
		* Maybe the original account still pays fees?
	* Maybe it should be limited to _just_ the granular permissions? Or just `TrustSet` things?
* Should the `NFTokenMinter` field be deprecated as a result?
	* Different reserve needs - no reserve needed for the `NFTokenMinter` field
		* However, this way would allow the minters to mint directly into your account
* Should the `TransactionAuth` and `DepositPreauth` transaction types operate exactly the same or is it okay for there to be a bit of a difference?

# Appendix

## Appendix A: Comparing with [XLS-49d](https://github.com/XRPLF/XRPL-Standards/discussions/144), Multiple Signer Lists

No multisig support, just a singular account has this permission (though that single account could have its own signer list). Both are useful for slightly different usecases; XLS-49d is useful when you want multiple signatures to guard certain features, while this proposal is useful when you want certain parties to have access to certain features.

The main difference is that with XLS-49d, the account controls the  signer list. With this proposal, the signer list is essentially self-governing, and it would take more reserves to set up such a signer list.

<!--
## Appendix B: FAQ

### B.1: 

