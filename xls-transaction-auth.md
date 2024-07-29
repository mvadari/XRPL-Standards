<pre>
Title:       <b>Account Permission Delegation</b>
Revision:    <b>1</b> (2024-05-29)

Author:      <a href="mailto:mvadari@ripple.com">Mayukha Vadari</a>

Affiliation: <a href="https://ripple.com">Ripple</a>
</pre>

# Account Permission Delegation

## Abstract

Authorizing trustlines is hard since the issuer needs to do it - hard to manage if the issuer has super locked up keys to ensure protection

Some folks want to have multiple `NFTokenMint`ers

Imagine if you could delegate some of those permissions to another account, without needing to give away total control of your account.

## 1. Overview

We propose creating one new ledger object and one new transaction type:
* `TransactionAuth` ledger object
* `TransactionAuth` transaction type

We also propose modifying the transaction common fields.

This feature will require an amendment, tentatively titled `featureTransactionAuth`.

### 1.1. Terminology

* **Permissions** - all the permissions you can have, could be a transaction type or something more granular

### 1.2. Permissions

The different types of permissions that can be supported are explained [here](https://gist.github.com/mvadari/a8d76f0c4e3aa54eb765f08bcacc5316).

## 2. On-Ledger Object: `TransactionAuth`

Follows the model of DepositPreauth

### 2.1. Fields

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`LedgerIndex`| ✔️|`string`|`Hash256`|The unique ID of the ledger object.|
|`LedgerEntryType`| ✔️|`string`|`UInt16`|The ledger object's type (`TransactionAuth`)|
|`Account`| ✔️|`string`|`AccountID`|The account that wants to authorize another account.|
|`Authorize`| ✔️|`string`|`AccountID`|The authorized account.|
|`Permissions`| ✔️|`string`|`STArray`|The transaction permissions that the account has access to.|

#### 2.1.1. `LedgerIndex`

Hash of `LedgerEntryType`, `Account`, and `Authorize`

#### 2.1.2. `Permissions`

Could be a transaction type value, or a more specific value. See other XLS spec about granular permissions.

## 3. Transaction: `TransactionAuth`

Follows the model of DepositPreauth

### 3.1. Fields

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`TransactionType`| ✔️|`string`|`UInt16`|The transaction type (`TransactionAuth`).|
|`Account`| ✔️|`string`|`AccountID`|The account that wants to authorize another account.|
|`Authorize`| |`string`|`AccountID`|The authorized account.|
|`Unauthorize`| |`string`|`AccountID`|The unauthorized account.|
|`Permissions`| ✔️|`string`|`STArray`|The transaction permissions that the account has access to.|

#### 3.1.1. 

### 3.2. Failure Conditions

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

* The `OnBehalfOf` account doesn't exist
* The `OnBehalfOf` account hasn't authorized the transaction's `Account` to send transactions on behalf of it
* The `OnBehalfOf` account hasn't authorized the transaction's `Account` to send this particular transaction type/granular permission on behalf of it

### 4.3. State Changes

The transaction succeeds as if the transaction was sent by the `OnBehalfOf` account. 

## 5. Examples

## 6. Invariants

## 7. Security

* Need to trust the account, especially if the powers the account has allow it to take funds from the account or charge reserves
* On the flip side, it allows accounts to delegate only certain permissions to other parties, instead of needing to give them access to the global signer list or even (if XLS-49d is enabled) access to a whole transaction

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

# Appendix

## Appendix A: Comparing with [XLS-49d](https://github.com/XRPLF/XRPL-Standards/discussions/144), Multiple Signer Lists

No multisig support, just a singular account has this permission (though that single account could have its own signer list). Both are useful for slightly different usecases; XLS-49d is useful when you want multiple signatures to guard certain features, while this proposal is useful when you want certain parties to have access to certain features.

The main difference is that with XLS-49d, the account controls the  signer list. With this proposal, the signer list is essentially self-governing, and it would take more reserves to set up such a signer list.


## Appendix A: FAQ

### A.1: 

