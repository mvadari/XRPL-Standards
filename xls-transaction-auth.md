<pre>
Title:       <b>Account Permission Delegation</b>
Revision:    <b>1</b> (2024-07-31)

Author:      <a href="mailto:mvadari@ripple.com">Mayukha Vadari</a>

Affiliation: <a href="https://ripple.com">Ripple</a>
</pre>

# Account Permission Delegation

## Abstract

This proposal introduces a delegated authorization mechanism to enhance the flexibility and usability of XRPL accounts.

Currently, critical issuer actions, such as authorizing trustlines, require direct control by the account's keys, hindering operational efficiency and complex use cases. By empowering account holders to selectively delegate specific permissions to other accounts, this proposal aims to enhance account usability without compromising security. This mechanism will unlock new possibilities for XRPL applications, such as multi-party workflows and advanced account management strategies.

## 1. Overview

We propose:
* Creating a `TransactionAuth` ledger object.
* Creating a `TransactionAuth` transaction type.

We also propose modifying the transaction common fields.

This feature will require an amendment, tentatively titled `featureTransactionAuth`.

### 1.1. Terminology

* **Permissions** - all the permissions you can have, could be a transaction type or something more granular. The different types of permissions that are supported are explained in [XLS-otherd](https://gist.github.com/mvadari/a8d76f0c4e3aa54eb765f08bcacc5316).

### 1.2. Basic Flow

Isaac, a token issuer, wants to set up his account to follow security best practices and separation of responsibilities. He wants Alice to handle token issuing and Bob to handle trustlines. He is also working with Kylie, a KYC provider, who he wants to be able to authorize trustlines but not have any other permissions (as she is an external party).

He can authorize:
* Alice's account for the `Payment` transaction permission.
* Bob's account for the `TrustSet` transaction permission.
* Kylie's account for the `TrustlineAuthorize` granular permission.

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
|`PreviousTxnID`|✔️|`string`|`Hash256`|The identifying hash of the transaction that most recently modified this object.|
|`PreviousTxnLgrSeqNumber`|✔️|`number`|`UInt32`|The index of the ledger that contains the transaction that most recently modified this object.|

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

This transaction works slightly differently from the `DepositPreauth` transaction type. Instead of using an `Unauthorize` field, an account is unauthorized by using an empty `Permissions` list.

### 3.2. Failure Conditions

* `Permissions` is too long.
* Any of the specified permissions are invalid.
* The account specified in `Authorize` doesn't exist.

### 3.3. State Changes

A `TransactionAuth` object will be created, modified, or deleted based on the provided fields.

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

In this example, Isaac is delegating the `Payment` permission to Alice, the `TrustSet` permission to Bob, and the `TrustlineAuthorize` permission to Kylie.

### 5.1. `TransactionAuth` Transaction

```typescript
{
    TransactionType: "TransactionAuth",
    Account: "rISAAC......",
    Authorize: "rALICE......",
    Permissions: [{Permission: {PermissionValue: "Payment"}}],
}
```
```typescript
{
    TransactionType: "TransactionAuth",
    Account: "rISAAC......",
    Authorize: "rBOB......",
    Permissions: [{Permission: {PermissionValue: "TrustSet"}}],
}
```
```typescript
{
    TransactionType: "TransactionAuth",
    Account: "rISAAC......",
    Authorize: "rKYLIE......",
    Permissions: [{Permission: {PermissionValue: "TrustlineAuthorize"}}],
}
```

_Note: the weird format of `Permissions`, with needing an internal object, is due to peculiarities in the [XRPL's Binary Format](https://xrpl.org/docs/references/protocol/binary-format)._

### 5.2. `TransactionAuth` Object

```typescript
{
    LedgerEntryType: "TransactionAuth",
    Account: "rISAAC......",
    Authorize: "rALICE......",
    Permissions: [{Permission: {PermissionValue: "Payment"}}],
}
```
```typescript
{
    LedgerEntryType: "TransactionAuth",
    Account: "rISAAC......",
    Authorize: "rBOB......",
    Permissions: [{Permission: {PermissionValue: "TrustSet"}}],
}
```
```typescript
{
    LedgerEntryType: "TransactionAuth",
    Account: "rISAAC......",
    Authorize: "rKYLIE......",
    Permissions: [{Permission: {PermissionValue: "TrustlineAuthorize"}}],
}
```

## 6. Invariants

* An account should never be able to send a transaction on behalf of another account without a valid `TransactionAuth` object.

## 7. Security

Delegating permissions to other accounts requires a high degree of trust, especially when the delegated account can potentially access funds (`Payment`s) or charge reserves (any transaction that can create objects). In addition, any account that has access to the entire `AccountSet`, `SignerListSet`, or `TransactionAuth` transactions can give themselves any permissions even if this was not originally part of the intention. Authorizing users for those transactions should have heavy warnings associated with it in tooling and UIs.

All tooling indicating whether an account has been blackholed will need to be updated to also check if `AccountSet`, `SignerListSet`, or `TransactionAuth` permissions have been delegated.

On the other hand, this mechanism also offers a granular approach to authorization, allowing accounts to selectively grant specific permissions without compromising overall account control. This approach provides a balance between security and usability, empowering account holders to manage their assets and interactions more effectively.

## n+1. Open Questions

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

