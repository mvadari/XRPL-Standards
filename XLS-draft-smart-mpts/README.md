<pre>
  title: Smart MPTs
  description: Allow Multi-Purpose Tokens to set conditions on when transfers are allowed.
  created: 2025-12-12
  author: Mayukha Vadari (@mvadari)
  status: Proposal
  requires: 33, 94, Confidential MPTs
  category: Amendment
</pre>

# Smart MPTs

## 1. Abstract

This proposal introduces a new [Smart Feature](https://dev.to/ripplexdev/a-proposed-vision-for-xrp-ledger-programmability-1gdj) that allows [Multi-Purpose Tokens](https://xrpl.org/docs/concepts/tokens/fungible-tokens/multi-purpose-tokens) to have custom rules, or **Transfer Conditions**, that must be met before they can be transferred.

The core of the system is a piece of custom WebAssembly code called a **Transfer Function**, which is attached to the token when it is first issued. When someone attempts a payment (or any other transfer of funds, such as a Check or Escrow), this code automatically runs a quick check (by running the Transfer Function) to determine if the transfer is permitted. If the check returns a "no," the transaction fails immediately.

## 2. Motivation

Regulated financial assets require a robust compliance framework on-chain to enable regulated participants to meet real-world regulatory requirements. Issuers of tokenized assets, or regulated institutions acting on their behalf, must have mechanisms in place to enforce transaction restrictions, track ownership, and apply necessary approvals for secondary transfers between two investors.

Some example use cases:

- **Investor Lockup**: Each asset amount received by an investor through a primary activity (e.g. subscription or distribution) must be subject to an individually applied lock-up period during which it cannot be transferred.
- **Minimum/Maximum Holdings**: Investors must maintain a non-zero asset balance within defined minimum and maximum thresholds. These thresholds may be applied universally or based on investor classification (e.g., EU or US investors).
- **Block Flowback**: Regulators may restrict securities transfers between investors based on their classification (e.g., jurisdiction or regulatory status), including rules that are direction-specific. For example, transfer rules may prohibit transfers from non-U.S. investors to U.S. investors to prevent flowback.
- **Force Full Transfer**: Asset issuers may enforce a rule requiring investors to transfer their entire holding of a given asset in a single transaction, thereby disallowing partial balance transfers.

## 3. Overview

This design follows the lead of [Smart Escrows](https://xls.xrpl.org/xls/XLS-0100-smart-escrows.html) and reuses many of the design paradigms established in that specification. This spec proposes:

- Modifying the `MPTokenIssuance` and `MPToken` ledger entries
- Modifying the `MPTokenIssuanceCreate`, `MPTokenIssuanceSet`, `Payment`, `EscrowFinish`, `ConfidentialSend`, `PaymentChannelClaim`, and `CheckCash` transactions

This feature will require an amendment, tentatively titled `SmartMPToken`.

This spec does not currently allow tokens with a transfer condition to be traded on the DEX.

Note: this spec assumes [XLS-94 (Dynamic MPTs)](https://xls.xrpl.org/xls/XLS-0094-dynamic-MPT.html) and [Confidential MPTs](https://github.com/XRPLF/XRPL-Standards/discussions/372) will both be implemented before this.

## 4. Ledger Entry: `MPTokenIssuance`

### 4.1. Fields

<details>
<summary>

As a reference, [here](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/mptokenissuance) are the existing fields for the `MPTokenIssuance` ledger entry.

</summary>

| Field Name          | JSON Type            | Internal Type | Required? | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| :------------------ | :------------------- | :------------ | :-------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Issuer`            | String               | AccountID     | Yes       | The address of the account that controls both the issuance amounts and characteristics of a particular fungible token.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `AssetScale`        | Number               | UInt8         | Yes       | Where to put the decimal place when displaying amounts of this MPT. More formally, the asset scale is a non-negative integer (0, 1, 2, …) such that one standard unit equals 10^(-scale) of a corresponding fractional unit. For example, if a US Dollar Stablecoin has an asset scale of _2_, then 1 unit of that MPT would equal 0.01 US Dollars. This indicates to how many decimal places the MPT can be subdivided. The default is `0`, meaning that the MPT cannot be divided into smaller than 1 unit.                                                                                               |
| `MaximumAmount`     | String - Number      | UInt64        | No        | The maximum number of MPTs that can exist at one time. If omitted, the maximum is currently limited to 2<sup>63</sup>-1.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `OutstandingAmount` | String - Number      | UInt64        | Yes       | The total amount of MPTs of this issuance currently in circulation. This value increases when the issuer sends MPTs to a non-issuer, and decreases whenever the issuer receives MPTs.                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `LockedAmount`      | String - Number      | UInt64        | No        | The amount of tokens currently locked up (for example, in escrow). This amount is already included in the `OutstandingAmount`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `TransferFee`       | Number               | UInt16        | Yes       | This value specifies the fee, in tenths of a basis point, charged by the issuer for secondary sales of the token, if such sales are allowed at all. Valid values for this field are between 0 and 50,000 inclusive. A value of 1 is equivalent to 1/10 of a basis point or 0.001%, allowing transfer rates between 0% and 50%. A `TransferFee` of 50,000 corresponds to 50%. The default value for this field is 0. Any decimals in the transfer fee are rounded down. The fee can be rounded down to zero if the payment is small. Issuers should make sure that their MPT's `AssetScale` is large enough. |
| `MPTokenMetadata`   | String - Hexadecimal | Blob          | Yes       | Arbitrary metadata about this issuance, in hex format. The limit for this field is 1024 bytes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `OwnerNode`         | String - Hexadecimal | UInt64        | Yes       | A hint indicating which page of the owner directory links to this entry, in case the directory consists of multiple pages.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `PreviousTxnID`     | String - Hexadecimal | UInt256       | Yes       | The identifying hash of the transaction that most recently modified this entry.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `PreviousTxnLgrSeq` | Number               | UInt32        | Yes       | The index of the ledger that contains the transaction that most recently modified this object.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `Sequence`          | Number               | UInt32        | Yes       | The `Sequence` (or `Ticket`) number of the transaction that created this issuance. This helps to uniquely identify the issuance and distinguish it from any other later MPT issuances created by this account.                                                                                                                                                                                                                                                                                                                                                                                              |

</details>

The existing fields still exist as-is. These fields are proposed as an addition:

| Field Name         | Constant? | Required? | Default Value | JSON Type | Internal Type | Description                                                                          |
| ------------------ | --------- | --------- | ------------- | --------- | ------------- | ------------------------------------------------------------------------------------ |
| `TransferFunction` | No        | No        | N/A           | `string`  | `Blob`        | A WASM module that governs whether an MPT transfer is allowed.                       |
| `Data`             | No        | No        | N/A           | `string`  | `Blob`        | User-defined extra data that can be accessed and modified by the `TransferFunction`. |

#### 4.1.1. `TransferFunction`

The compiled WASM code included in this field must contain a function named `transfer` that takes no parameters and returns a signed integer (int32). If the function returns a value greater than 0, the transfer is allowed. If the function returns 0 or a negative number, the transfer is disallowed.

The function is triggered on every MPT transfer transaction that involves this MPT issuance.

See [XLS-102](https://xls.xrpl.org/xls/XLS-0102-wasm-vm.html) for detailed specifications of each host function and their gas costs.

### 4.2. Reserves

An `MPTokenIssuance` object with a `TransferFunction` will cost 1 additional object reserve per 500 bytes (beyond the first 500 bytes, which are included in the first object reserve).

## 5. Ledger Entry: `MPToken`

### 5.1. Fields

<details>
<summary>

As a reference, [here](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/mptoken) are the existing fields for the `MPToken` ledger entry.

</summary>

| Field Name          | JSON Type            | Internal Type | Required? | Description                                                                                                                |
| :------------------ | :------------------- | :------------ | :-------- | :------------------------------------------------------------------------------------------------------------------------- |
| `Account`           | String - Address     | AccountID     | Yes       | The owner (holder) of these MPTs.                                                                                          |
| `MPTokenIssuanceID` | String - Hexadecimal | UInt192       | Yes       | The `MPTokenIssuance` identifier.                                                                                          |
| `MPTAmount`         | String - Number      | UInt64        | Yes       | The amount of tokens currently held by the owner. The minimum is 0 and the maximum is 2<sup>63</sup>-1.                    |
| `LockedAmount`      | String - Number      | UInt64        | No        | The amount of tokens currently locked up (for example, in escrow).                                                         |
| `PreviousTxnID`     | String - Hash        | UInt256       | Yes       | The identifying hash of the transaction that most recently modified this entry.                                            |
| `PreviousTxnLgrSeq` | Number               | UInt32        | Yes       | The sequence of the ledger that contains the transaction that most recently modified this object.                          |
| `OwnerNode`         | String               | UInt64        | Yes       | A hint indicating which page of the owner directory links to this entry, in case the directory consists of multiple pages. |

</details>

The existing fields still exist as-is. These fields are proposed as an addition:

| Field Name | Constant? | Required? | Default Value | JSON Type | Internal Type | Description                                                                              |
| ---------- | --------- | --------- | ------------- | --------- | ------------- | ---------------------------------------------------------------------------------------- |
| `Data`     | No        | No        | N/A           | `string`  | `Blob`        | WASM dev-defined extra data that can be accessed and modified by the `TransferFunction`. |

## 6. Transaction: `MPTokenIssuanceCreate`

### 6.1. Fields

<details>
<summary>

As a reference, [here](https://xrpl.org/docs/references/protocol/transactions/types/mptokenissuancecreate) are the existing fields for the `MPTokenIssuanceCreate` transaction.

</summary>

| Field             | JSON Type            | Internal Type | Required? | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| :---------------- | :------------------- | :------------ | :-------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AssetScale`      | Number               | UInt8         | No        | Where to put the decimal place when displaying amounts of this MPT. More formally, the asset scale is a non-negative integer (0, 1, 2, …) such that one standard unit equals 10^(-scale) of a corresponding fractional unit. For example, if a US Dollar Stablecoin has an asset scale of _2_, then 1 unit of that MPT would equal 0.01 US Dollars. This indicates to how many decimal places the MPT can be subdivided. If omitted, the default is 0, meaning that the MPT cannot be divided into smaller than 1 unit. |
| `TransferFee`     | Number               | UInt16        | No        | The value specifies the fee to charged by the issuer for secondary sales of the Token, if such sales are allowed. Valid values for this field are between 0 and 50,000 inclusive, allowing transfer rates of between 0.000% and 50.000% in increments of 0.001. The field _must not_ be present if the tfMPTCanTransfer flag is not set. If it is, the transaction should fail and a fee should be claimed.                                                                                                             |
| `MaximumAmount`   | String - Number      | UInt64        | No        | The maximum asset amount of this token that can ever be issued, as a base-10 number encoded as a string. The current default maximum limit is 9,223,372,036,854,775,807 (2^63-1). _This limit may increase in the future. If an upper limit is required, you must specify this field._                                                                                                                                                                                                                                  |
| `MPTokenMetadata` | String - Hexadecimal | Blob          | No        | Arbitrary metadata about this issuance. The limit for this field is 1024 bytes. By convention, the metadata should decode to JSON data describing what the MPT represents. The [XLS-89 specification](https://xls.xrpl.org/xls/XLS-0089-multi-purpose-token-metadata-schema.html) defines a recommended format for metadata.                                                                                                                                                                                            |
| `MutableFlags`    | Number               | UInt32        | No        | Bits in `MutableFlags` indicate specific fields or flags may be modified after issuance. Introduced in XLS-94.                                                                                                                                                                                                                                                                                                                                                                                                          |

</details>

The existing fields still exist as-is, with `MutableFlags` having some additional possible values. These fields are proposed as an addition:

| Field Name         | Required? | JSON Type | Internal Type | Description                                                                          |
| ------------------ | --------- | --------- | ------------- | ------------------------------------------------------------------------------------ |
| `TransferFunction` | No        | `string`  | `Blob`        | A WASM module that governs whether an MPT transfer is allowed.                       |
| `Data`             | No        | `string`  | `Blob`        | User-defined extra data that can be accessed and modified by the `TransferFunction`. |

#### 6.1.1. Mutable Flags

Mutable flags are proposed in [XLS-94](https://xls.xrpl.org/xls/XLS-0094-dynamic-MPT.html). These are the current valid values:

| Flag Name                    |   Hex Value   | Decimal Value | Description                                                                                  |
| ---------------------------- | :-----------: | :-----------: | -------------------------------------------------------------------------------------------- |
| [Reserved]                   | `0x00000001`  |       1       | [Reserved; To align with `Flags` values, the `MutableFlags` value starts from `0x00000002`.] |
| `tmfMPTCanMutateCanLock`     | ️`0x00000002` |       2       | Indicates flag `lsfMPTCanLock` can be changed                                                |
| `tmfMPTCanMutateRequireAuth` | ️`0x00000004` |       4       | Indicates flag `lsfMPTRequireAuth` can be changed                                            |
| `tmfMPTCanMutateCanEscrow`   | `0x00000008`  |       8       | Indicates flag `lsfMPTCanEscrow` can be changed                                              |
| `tmfMPTCanMutateCanTrade`    | `0x00000010`  |      16       | Indicates flag `lsfMPTCanTrade` can be changed                                               |
| `tmfMPTCanMutateCanTransfer` | ️`0x00000020` |      32       | Indicates flag `lsfMPTCanTransfer` can be changed                                            |
| `tmfMPTCanMutateCanClawback` | ️`0x00000040` |      64       | Indicates flag `lsfMPTCanClawback` can be changed                                            |
| `tmfMPTCanMutateMetadata`    | `0x00010000`  |     65536     | Allows field `MPTokenMetadata` to be modified                                                |
| `tmfMPTCanMutateTransferFee` | `0x00020000`  |    131072     | Allows field `TransferFee` to be modified                                                    |

This spec proposes the following additions:

| Flag Name                         |  Hex Value   | Decimal Value | Description                                    |
| --------------------------------- | :----------: | :-----------: | ---------------------------------------------- |
| `tmfMPTCanMutateTransferFunction` | `0x00040000` |    262144     | Allows field `TransferFunction` to be modified |
| `tmfMPTCanMutateData`             | `0x00080000` |    524288     | Allows field `Data` to be modified             |

### 6.2. Transaction Fee

An `MPTokenIssuanceCreate` transaction with a `TransferFunction` costs an additional 100 drops ($base\_fee * 10$) + 5 drops per byte in the `TransferFunction`.

### 6.3. Failure Conditions

The existing failure conditions are still included, plus:

- `TransferFunction` is not valid WASM code or is invalid in some other way (e.g., using a nonexistent host function).
- `TransferFunction` does not abide by the ABI (must have a `transfer()` function that takes no parameters and returns i32).
- The length of `TransferFunction` exceeds the size limit (100 KB, as specified in FeeSettings).
- The `Data` field length exceeds the size limit (4 KB, as specified in FeeSettings).
- `TransferFunction` is included, but `tfMPTCanTransfer` is not.
- `TransferFunction` and `tfMPTCanTrade` are both included.

### 6.4. State Changes

If the transaction is successful:

- The `MPTokenIssuance` object is created as normal, with the `TransferFunction` and `Data` fields, if applicable.

## 7. Transaction: `MPTokenIssuanceSet`

### 7.1. Fields

<details>
<summary>

As a reference, [here](https://xls.xrpl.org/xls/XLS-0094-dynamic-MPT.html#4-mptokenissuanceset-transaction-update) are the existing fields for the `MPTokenIssuanceSet` transaction, with XLS-94 in effect.

</summary>

| Field               | JSON Type            | Internal Type | Required? | Description                                                                                                                                                                                     |
| :------------------ | :------------------- | :------------ | :-------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `MPTokenIssuanceID` | String - Hexadecimal | UInt192       | Yes       | The identifier of the `MPTokenIssuance` to update.                                                                                                                                              |
| `Holder`            | String - Address     | AccountID     | No        | An individual token holder. If provided, apply changes to the given holder's balance of the given MPT issuance. If omitted, apply to all accounts holding the given MPT issuance.               |
| `MPTokenMetadata`   |                      | `string`      | Blob      | New metadata to replace the existing value. The transaction will be rejected if `lsmfMPTCanMutateMetadata` was not set in `MutableFlags`. Setting an empty `MPTokenMetadata` removes the field. |
| `TransferFee`       |                      | `number`      | UInt16    | New transfer fee value. The transaction will be rejected if `lsmfMPTCanMutateTransferFee` was not set in `MutableFlags`. Setting `TransferFee` to zero removes the field.                       |
| `MutableFlags`      |                      | `number`      | UInt32    | Set or clear the flags which were marked as mutable.                                                                                                                                            |

</details>

The existing fields still exist as-is. These fields are proposed as additions:

| Field Name         | Required? | JSON Type | Internal Type | Description                                                                                                                                                                                                   |
| ------------------ | --------- | --------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TransferFunction` | No        | `string`  | `Blob`        | A WASM module that governs whether an MPT transfer is allowed. Can only be modified if the MPTIssuance has the `lsmfTransferFunctionMutable` flag set. Setting an empty `TransferFunction` removes the field. |
| `Data`             | No        | `string`  | `Blob`        | User-defined extra data that can be accessed and modified by the `TransferFunction`. Can only be modified if the MPTIssuance has the `lsmfDataMutable` flag set. Setting an empty `Data` removes the field.   |

### 7.2. Transaction Fee

An `MPTokenIssuanceSet` transaction with a `TransferFunction` costs an additional 100 drops ($base\_fee * 10$) + 5 drops per byte in the `TransferFunction`.

### 7.3. Failure Conditions

The existing failure conditions are still included, plus:

- `TransferFunction` is not valid WASM code or is invalid in some other way (e.g., using a nonexistent host function).
- `TransferFunction` does not abide by the ABI (must have a `transfer()` function that takes no parameters and returns i32).
- The length of `TransferFunction` exceeds the size limit (100 KB, as specified in FeeSettings).
- The length of `Data` exceeds the size limit (4 KB, as specified in FeeSettings).
- `TransferFunction` is being modified but the MPTIssuance does not have the `TransferFunctionMutable` flag set.
- `Data` is being modified but the MPTIssuance does not have the `DataMutable` flag set.
- `tmfMPTClearCanTrade` is set but `Data` is not cleared.

### 7.4. State Changes

If the transaction is successful:

- The `MPTokenIssuance` object is updated with the new `TransferFunction` and/or `Data` fields, if applicable.

## 8. Transaction: `Payment`

### 8.1. Fields

<details>
<summary>

As a reference, [here](http://xrpl.org/docs/references/protocol/transactions/types/payment) are the existing fields for the `Payment` transaction.

</summary>

| Field            | JSON Type            | Internal Type | Required?   | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| :--------------- | :------------------- | :------------ | :---------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Amount`         | Currency Amount      | Amount        | API v1: Yes | Alias to `DeliverMax`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `CredentialIDs`  | Array of Strings     | Vector256     | No          | Set of Credentials to authorize a deposit made by this transaction. Each member of the array must be the ledger entry ID of a Credential entry in the ledger.                                                                                                                                                                                                                                                                                                                                               |
| `DeliverMax`     | Currency Amount      | Amount        | Yes         | The maximum amount of currency to deliver. Partial payments can deliver less than this amount and still succeed; other payments fail unless they deliver the exact amount.                                                                                                                                                                                                                                                                                                                                  |
| `DeliverMin`     | Currency Amount      | Amount        | No          | Minimum amount of destination currency this transaction should deliver. Only valid if this is a partial payment.                                                                                                                                                                                                                                                                                                                                                                                            |
| `Destination`    | String - Address     | AccountID     | Yes         | The account receiving the payment.                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `DestinationTag` | Number               | UInt32        | No          | Arbitrary tag that identifies the reason for the payment to the destination, or a hosted recipient to pay.                                                                                                                                                                                                                                                                                                                                                                                                  |
| `DomainID`       | String - Hash        | UInt256       | No          | The ledger entry ID of a permissioned domain. If this is a cross-currency payment, only use the corresponding [permissioned DEX](https://xrpl.org/docs/concepts/tokens/decentralized-exchange/permissioned-dexes) to convert currency. Both the sender and the recipient must have valid credentials that grant access to the specified domain. This field has no effect if the payment is not cross-currency.                                                                                              |
| `InvoiceID`      | String - Hexadecimal | UInt256       | No          | Arbitrary 256-bit value representing a specific reason or identifier for this payment.                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `Paths`          | Array of path arrays | PathSet       | No          | _(Auto-fillable)_ Array of [payment paths](https://xrpl.org/docs/concepts/tokens/fungible-tokens/paths) to be used for this transaction. Must be omitted for XRP-to-XRP transactions.                                                                                                                                                                                                                                                                                                                       |
| `SendMax`        | Currency Amount      | Amount        | No          | Highest amount of source currency this transaction is allowed to cost, including [transfer fees](https://xrpl.org/docs/concepts/tokens/fungible-tokens/transfer-fees), exchange rates, and [slippage](http://en.wikipedia.org/wiki/Slippage_%28finance%29). Does not include the [XRP destroyed as a cost for submitting the transaction](https://xrpl.org/docs/concepts/transactions/transaction-cost). Must be supplied for cross-currency/cross-issue payments. Must be omitted for XRP-to-XRP payments. |

</details>

The existing fields still exist. This field is proposed as an addition:

| Field Name             | Required? | JSON Type | Internal Type | Description                                                                                                                                           |
| :--------------------- | :-------- | :-------- | :------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ComputationAllowance` | No        | `number`  | `UInt32`      | The amount of gas the user is willing to pay for the execution of the `TransferFunction`. Required if the MPTIssuance has a `TransferFunction` field. |

### 8.2. Failure Conditions

The existing failure conditions are still included, plus:

- `ComputationAllowance` is included, but the `MPTIssuance` doesn't have a `TransferFunction`
- The `MPTIssuance` has a `TransferFunction`, but a `ComputationAllowance` isn't included
- The `ComputationAllowance` provided is not enough gas to complete the execution of the `TransferFunction`
- The `TransferFunction` returns 0 or a negative number (transfer is disallowed)

### 8.3. Metadata Changes

There are two additional metadata fields:

| Field Name       | Validated? | Always Present? | Type   | Description                                                                                                                                 |
| :--------------- | :--------- | :-------------- | :----- | :------------------------------------------------------------------------------------------------------------------------------------------ |
| `GasUsed`        | Yes        | Conditional     | UInt32 | The amount of gas actually used by the execution of the `TransferFunction`. Only present if the MPTIssuance has a `TransferFunction` field. |
| `WasmReturnCode` | Yes        | Conditional     | Int32  | The integer code returned by the `TransferFunction`. Only present if the MPTIssuance has a `TransferFunction` field.                        |

## 9. Transaction: `EscrowFinish`

Similar changes will apply to `EscrowFinish`, `ConfidentialSend`, `PaymentChannelClaim`, `CheckCash`

### 9.1. Fields

<details>
<summary>

As a reference, [here](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/mptokenissuance) are the existing fields for the `MPTokenIssuance` transaction.

</summary>

| Field Name          | JSON Type            | Internal Type | Required? | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| :------------------ | :------------------- | :------------ | :-------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Issuer`            | String               | AccountID     | Yes       | The address of the account that controls both the issuance amounts and characteristics of a particular fungible token.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `AssetScale`        | Number               | UInt8         | Yes       | Where to put the decimal place when displaying amounts of this MPT. More formally, the asset scale is a non-negative integer (0, 1, 2, …) such that one standard unit equals 10^(-scale) of a corresponding fractional unit. For example, if a US Dollar Stablecoin has an asset scale of _2_, then 1 unit of that MPT would equal 0.01 US Dollars. This indicates to how many decimal places the MPT can be subdivided. The default is `0`, meaning that the MPT cannot be divided into smaller than 1 unit.                                                                                               |
| `MaximumAmount`     | String - Number      | UInt64        | No        | The maximum number of MPTs that can exist at one time. If omitted, the maximum is currently limited to 2<sup>63</sup>-1.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `OutstandingAmount` | String - Number      | UInt64        | Yes       | The total amount of MPTs of this issuance currently in circulation. This value increases when the issuer sends MPTs to a non-issuer, and decreases whenever the issuer receives MPTs.                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `LockedAmount`      | String - Number      | UInt64        | No        | The amount of tokens currently locked up (for example, in escrow). This amount is already included in the `OutstandingAmount`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `TransferFee`       | Number               | UInt16        | Yes       | This value specifies the fee, in tenths of a basis point, charged by the issuer for secondary sales of the token, if such sales are allowed at all. Valid values for this field are between 0 and 50,000 inclusive. A value of 1 is equivalent to 1/10 of a basis point or 0.001%, allowing transfer rates between 0% and 50%. A `TransferFee` of 50,000 corresponds to 50%. The default value for this field is 0. Any decimals in the transfer fee are rounded down. The fee can be rounded down to zero if the payment is small. Issuers should make sure that their MPT's `AssetScale` is large enough. |
| `MPTokenMetadata`   | String - Hexadecimal | Blob          | Yes       | Arbitrary metadata about this issuance, in hex format. The limit for this field is 1024 bytes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `OwnerNode`         | String - Hexadecimal | UInt64        | Yes       | A hint indicating which page of the owner directory links to this entry, in case the directory consists of multiple pages.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `PreviousTxnID`     | String - Hexadecimal | UInt256       | Yes       | The identifying hash of the transaction that most recently modified this entry.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `PreviousTxnLgrSeq` | Number               | UInt32        | Yes       | The index of the ledger] that contains the transaction that most recently modified this object.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `Sequence`          | Number               | UInt32        | Yes       | The `Sequence` (or `Ticket`) number of the transaction that created this issuance. This helps to uniquely identify the issuance and distinguish it from any other later MPT issuances created by this account.                                                                                                                                                                                                                                                                                                                                                                                              |

</details>

The existing fields still exist as-is. This field is proposed as an addition:

| Field Name             | Required? | JSON Type | Internal Type | Description                                                                                                                                           |
| :--------------------- | :-------- | :-------- | :------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ComputationAllowance` | No        | `number`  | `UInt32`      | The amount of gas the user is willing to pay for the execution of the `TransferFunction`. Required if the MPTIssuance has a `TransferFunction` field. |

### 9.2. Failure Conditions

The existing failure conditions are still included, plus:

- `ComputationAllowance` is included, but the `MPTIssuance` doesn't have a `TransferFunction`
- The `MPTIssuance` has a `TransferFunction`, but a `ComputationAllowance` isn't included
- The `ComputationAllowance` provided is not enough gas to complete the execution of the `TransferFunction`
- The `TransferFunction` returns 0 or a negative number (transfer is disallowed)

### 9.3. Metadata Changes

There are two additional metadata fields:

| Field Name       | Validated? | Always Present? | Type   | Description                                                                                                                                 |
| :--------------- | :--------- | :-------------- | :----- | :------------------------------------------------------------------------------------------------------------------------------------------ |
| `GasUsed`        | Yes        | Conditional     | UInt32 | The amount of gas actually used by the execution of the `TransferFunction`. Only present if the MPTIssuance has a `TransferFunction` field. |
| `WasmReturnCode` | Yes        | Conditional     | Int32  | The integer code returned by the `TransferFunction`. Only present if the MPTIssuance has a `TransferFunction` field.                        |

## 10. Transaction: `PaymentChannelClaim`

<details>
<summary>

As a reference, [here](https://xrpl.org/docs/references/protocol/transactions/types/paymentchannelclaim) are the existing fields for the `PaymentChannelClaim` transaction.

</summary>

| Field           | JSON Type            | Internal Type | Required? | Description                                                                                                                                                                                                                                                                                                                                                                                     |
| :-------------- | :------------------- | :------------ | :-------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Amount`        | Currency Amount      | Amount        | No        | The amount of XRP, in drops, authorized by the `Signature`. This must match the amount in the signed message. This is the cumulative amount of XRP that can be dispensed by the channel, including XRP previously redeemed. Must be provided except when closing the channel.                                                                                                                   |
| `Balance`       | Currency Amount      | Amount        | No        | Total amount of XRP, in drops, delivered by this channel after processing this claim. Required to deliver XRP. Must be more than the total amount delivered by the channel so far, but not greater than the `Amount` of the signed claim. Must be provided except when closing the channel.                                                                                                     |
| `Channel`       | String - Hexadecimal | UInt256       | Yes       | The unique ID of the channel.                                                                                                                                                                                                                                                                                                                                                                   |
| `CredentialIDs` | Array of Strings     | Vector256     | No        | Set of credentials to authorize a deposit made by this transaction. Each member of the array must be the ledger entry ID of a Credential entry in the ledger.                                                                                                                                                                                                                                   |
| `PublicKey`     | String - Hexadecimal | Blob          | No        | The public key used for the signature. This must match the `PublicKey` stored in the ledger for the channel. Required unless the sender of the transaction is the source address of the channel and the `Signature` field is omitted. (The transaction includes the public key so that `rippled` can check the validity of the signature before trying to apply the transaction to the ledger.) |
| `Signature`     | String - Hexadecimal | Blob          | No        | The signature of this claim. The signed message contains the channel ID and the amount of the claim. Required unless the sender of the transaction is the source address of the channel.                                                                                                                                                                                                        |

</details>

### 10.1. Fields

The existing fields still exist as-is. This field is proposed as an addition:

| Field Name             | Required? | JSON Type | Internal Type | Description                                                                                                                                           |
| :--------------------- | :-------- | :-------- | :------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ComputationAllowance` | No        | `number`  | `UInt32`      | The amount of gas the user is willing to pay for the execution of the `TransferFunction`. Required if the MPTIssuance has a `TransferFunction` field. |

### 10.2. Failure Conditions

The existing failure conditions are still included, plus:

- `ComputationAllowance` is included, but the `MPTIssuance` doesn't have a `TransferFunction`
- The `MPTIssuance` has a `TransferFunction`, but a `ComputationAllowance` isn't included
- The `ComputationAllowance` provided is not enough gas to complete the execution of the `TransferFunction`
- The `TransferFunction` returns 0 or a negative number (transfer is disallowed)

### 10.3. Metadata Changes

There are two additional metadata fields:

| Field Name       | Validated? | Always Present? | Type   | Description                                                                                                                                 |
| :--------------- | :--------- | :-------------- | :----- | :------------------------------------------------------------------------------------------------------------------------------------------ |
| `GasUsed`        | Yes        | Conditional     | UInt32 | The amount of gas actually used by the execution of the `TransferFunction`. Only present if the MPTIssuance has a `TransferFunction` field. |
| `WasmReturnCode` | Yes        | Conditional     | Int32  | The integer code returned by the `TransferFunction`. Only present if the MPTIssuance has a `TransferFunction` field.                        |

## 11. Transaction: `CheckCash`

### 11.1. Fields

<details>
<summary>

As a reference, [here](https://xrpl.org/docs/references/protocol/transactions/types/checkcash) are the existing fields for the `CheckCash` transaction.

</summary>

| Field        | JSON Type       | Internal Type | Description                                                                                                                                                                                                                     |
| :----------- | :-------------- | :------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `CheckID`    | String          | UInt256       | The ID of the Check ledger object to cash, as a 64-character hexadecimal string.                                                                                                                                                |
| `Amount`     | Currency Amount | Amount        | _(Optional)_ Redeem the Check for exactly this amount, if possible. The currency must match that of the `SendMax` of the corresponding CheckCreate transaction. You must provide either this field or `DeliverMin`.             |
| `DeliverMin` | Currency Amount | Amount        | _(Optional)_ Redeem the Check for at least this amount and for as much as possible. The currency must match that of the `SendMax` of the corresponding CheckCreate transaction. You must provide either this field or `Amount`. |

</details>

The existing fields still exist as-is. This field is proposed as an addition:

| Field Name             | Required? | JSON Type | Internal Type | Description                                                                                                                                           |
| :--------------------- | :-------- | :-------- | :------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ComputationAllowance` | No        | `number`  | `UInt32`      | The amount of gas the user is willing to pay for the execution of the `TransferFunction`. Required if the MPTIssuance has a `TransferFunction` field. |

### 11.2. Failure Conditions

The existing failure conditions are still included, plus:

- `ComputationAllowance` is included, but the `MPTIssuance` doesn't have a `TransferFunction`
- The `MPTIssuance` has a `TransferFunction`, but a `ComputationAllowance` isn't included
- The `ComputationAllowance` provided is not enough gas to complete the execution of the `TransferFunction`
- The `TransferFunction` returns 0 or a negative number (transfer is disallowed)

### 11.3. Metadata Changes

There are two additional metadata fields:

| Field Name       | Validated? | Always Present? | Type   | Description                                                                                                                                 |
| :--------------- | :--------- | :-------------- | :----- | :------------------------------------------------------------------------------------------------------------------------------------------ |
| `GasUsed`        | Yes        | Conditional     | UInt32 | The amount of gas actually used by the execution of the `TransferFunction`. Only present if the MPTIssuance has a `TransferFunction` field. |
| `WasmReturnCode` | Yes        | Conditional     | Int32  | The integer code returned by the `TransferFunction`. Only present if the MPTIssuance has a `TransferFunction` field.                        |

## 12. Transaction: `ConfidentialSend`

`ConfidentialSend` is a transaction included in the [Confidential MPT amendment proposal](https://github.com/XRPLF/XRPL-Standards/discussions/372). It allows a confidential transfer of MPT value between accounts while keeping the amount hidden.

### 12.1. Fields

<details>
<summary>

As a reference, [here](https://github.com/XRPLF/XRPL-Standards/discussions/372) are the existing fields for the `ConfidentialSend` transaction.

</summary>

| Field                      | Required    | JSON Type | Internal Type          | Description                                                                                                                                         |
| :------------------------- | :---------- | :-------- | :--------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------- |
| MPTokenIssuanceID          | Yes         | String    | UInt256                | The unique identifier for the MPT being converted.                                                                                                  |
| Receiver                   | Yes         | String    | AccountID              | The recipient's account address.                                                                                                                    |
| EncryptedAmountForSender   | Yes         | Object    | Struct (EC Point Pair) | EC-ElGamal ciphertext for the sender, representing a debit to their balance.                                                                        |
| EncryptedAmountForReceiver | Yes         | Object    | Struct (EC Point Pair) | EC-ElGamal ciphertext for the receiver, representing a credit to their balance.                                                                     |
| EncryptedAmountForIssuer   | Yes         | Object    | Struct (EC Point Pair) | EC-ElGamal ciphertext for the issuer, used for auditing purposes.                                                                                   |
| EncryptedAmountForAuditor  | Optional    | Object    | Struct (EC Point Pair) | EC-ElGamal ciphertext for a designated auditor, if an audit policy is active.                                                                       |
| AuditorPublicKey           | Conditional | String    | Blob (EC Point)        | The auditor's public key. Required if EncryptedAmountForAuditor is present.                                                                         |
| ZKProof                    | Yes         | Object    | Blob                   | A ZKP that validates the entire transaction, ensuring the amount is positive, within the sender's balance, and that all ciphertexts are consistent. |

</details>

The existing fields still exist as-is. This field is proposed as an addition:

| Field Name             | Required? | JSON Type | Internal Type | Description                                                                                                                                           |
| :--------------------- | :-------- | :-------- | :------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ComputationAllowance` | No        | `number`  | `UInt32`      | The amount of gas the user is willing to pay for the execution of the `TransferFunction`. Required if the MPTIssuance has a `TransferFunction` field. |

### 12.2. Failure Conditions

The existing failure conditions are still included, plus:

- `ComputationAllowance` is included, but the `MPTIssuance` doesn't have a `TransferFunction`
- The `MPTIssuance` has a `TransferFunction`, but a `ComputationAllowance` isn't included
- The `ComputationAllowance` provided is not enough gas to complete the execution of the `TransferFunction`
- The `TransferFunction` returns 0 or a negative number (transfer is disallowed)

### 12.3. Metadata Changes

There are two additional metadata fields:

| Field Name       | Validated? | Always Present? | Type   | Description                                                                                                                                 |
| :--------------- | :--------- | :-------------- | :----- | :------------------------------------------------------------------------------------------------------------------------------------------ |
| `GasUsed`        | Yes        | Conditional     | UInt32 | The amount of gas actually used by the execution of the `TransferFunction`. Only present if the MPTIssuance has a `TransferFunction` field. |
| `WasmReturnCode` | Yes        | Conditional     | Int32  | The integer code returned by the `TransferFunction`. Only present if the MPTIssuance has a `TransferFunction` field.                        |

## 13. Host Functions

The following host functions will be added to the WASM VM, only enabled for Smart MPTs:

| Function Signature                                                                                                       | Description                                                                           | Gas Cost |
| :----------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------ | :------- |
| `update_user_data(` <br> `account_ptr: i32,` <br> `account_len: i32,` <br> `data_ptr: i32,` <br> `data_len: i32` <br>`)` | Update the `Data` field in the `MPToken` object corresponding to the provided holder. | 50       |
| `update_data(` <br> `data_ptr: i32,` <br> `data_len: i32` <br>`)`                                                        | Update the `Data` field in `MPTokenIssuance` object.                                  | 50       |

## 14. Rationale

The core motivation is to move critical compliance functions from off-chain systems directly onto the XRPL, making Multi-Purpose Tokens (MPTs) a viable standard for tokenized assets like securities.

- **Enforcing Compliance On-Chain:** The main design feature, the **Transfer Function**, is a piece of custom WebAssembly (WASM) code. This directly addresses the need for **Transfer Conditions** that must be enforced for _every_ transfer event, regardless of the transaction type.
- **Easy Auditability:** The `transfer()` function signature (no parameters, returns `i32`) enforces a simple, auditable **Allowed/Disallowed** outcome, protecting the integrity of the ledger. A return value $>0$ permits the transfer, while $\le 0$ rejects it.

### 14.1. Alternate Designs Considered

- **Native Ledger Rules:** An alternative would have been to implement a limited set of compliance rules (e.g., a simple whitelist/blacklist) as native ledger fields. This was rejected because it would be impossible to foresee and implement the vast and evolving spectrum of global regulatory requirements. The WASM-based Transfer Function provides a future-proof, customizable solution.
- **Off-Ledger Compliance Checks:** Relying on external, off-chain systems to pre-validate transactions was considered. This was rejected as it breaks the principle of a trustless, decentralized ledger. By enforcing the Transfer Conditions directly on-chain, the system ensures that the transfer rule is executed atomically and immutably with the transaction itself.
- **Smart Contracts (General Purpose):** Leveraging a separate, general-purpose smart contract platform was deemed unnecessary overhead. The Smart MPTs design integrates the transfer logic directly into the MPT issuance entry, making it a tightly coupled feature of the token standard rather than an external dependency.

### 14.2. Related Work

- **Other Blockchains:** Many modern blockchain platforms, such as **Ethereum (ERC-20)** and **Solana (Token Extensions)**, have mechanisms for custom token logic, often through a general-purpose smart contract platform. Smart MPTs are similar to the _transfer hook_ pattern seen in some of these systems but are specifically tailored to the XRPL's MPT standard, resulting in a more gas-efficient and integrated solution for compliance-focused assets.

### 14.3. Objections and Concerns

During the proposal's discussion, two primary concerns were raised:

1. **DEX Compatibility:** A key objection was the decision to **disallow tokens with a Transfer Function from trading on the Decentralized Exchange (DEX)**. This was a necessary trade-off. Enforcing complex, custom-coded transfer conditions on every DEX offer and trade execution would significantly increase the complexity, transaction cost, and risk of the DEX engine. The current design prioritizes compliance for regulated OTC (Over-The-Counter) transfers over compatibility with the DEX's open market trading, which is primarily intended for un-restricted assets.
2. **Transaction Fee and WASM Size:** Concerns were raised regarding the potential for large, expensive Transfer Functions. The specification mitigates this by:
   - Introducing a transaction fee that is directly proportional to the size of the Transfer Function ($5$ drops per byte).
   - Enforcing a strict size limit on the WASM code (100 KB) and the associated `Data` field (4 KB), as defined in FeeSettings, to ensure the feature remains lightweight and scalable for the ledger.

## 15. Security Considerations

The access control for managing the MPToken issuance and transfer is **highly restricted to ensure security and predictable behavior**. Specifically:

- Only the token issuer will be able to modify the `TransferFunction`.
- Only the token issuer and the WASM code in the `TransferFunction` will be able to modify the `Data` on the `MPTokenIssuance` object.
- Only the WASM code in the `TransferFunction` will be able to modify the `Data` on the `MPToken` object.

This design incorporates a key security measure by using a fully sandboxed WASM Virtual Machine (VM) (as outlined in detail in [XLS-102](https://xls.xrpl.org/xls/XLS-0102-wasm-vm.html#6-security)). The WASM VM's write access is strictly limited to the two specified `Data` fields (`MPTokenIssuance` and `MPToken` objects), which fundamentally prevents the code from initiating unauthorized transactions or "stealing money". However, users should be aware of a potential risk: a bug or error in the developer-written WASM code could inadvertently "brick" a token, preventing it from being traded or transferred as intended.

<!--
## 16. n+1. Open Questions

### 16.1. TODOs

- Determine if the transfer check should happen on create (e.g. `EscrowCreate`) or finish (e.g. `EscrowFinish`).
- Should the WASM code also not be able to change `Data` if `tmfMPTCanMutateData` is not enabled?
- -->

# Appendix

## Appendix A: FAQ

### A.1: Why is this only for MPTs? Why not also support IOUs?

It's a lot easier to integrate this paradigm in MPTs vs IOUs - there's an issuance object where this data can be stored, there's a definite issuer, and there's no additional complication of rippling.

### A2: Can Smart MPTs be traded on the DEX?

No.
