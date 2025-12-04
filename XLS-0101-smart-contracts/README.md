<pre>
  title: XRPL Smart Contracts
  description: An L1 native implementation of Smart Contracts on the XRP Ledger
  created: 2025-11-05
  Author: Mayukha Vadari (@mvadari), Denis Angell (@dangell7)
  status: Proposal
  category: Amendment
</pre>

# XRPL Smart Contracts

## Abstract

This document is a formal design of a smart contract system for the XRPL, which takes inspiration from several existing smart contract systems (including Xahau's Hooks and the EVM).

Some sample use cases (a far-from-exhaustive list):

- Integration with a new bridging protocol
- A new DeFi protocol (e.g. derivatives or perpetuals)
- Issued token staking rewards

_Note: This document has been updated to reflect the actual implementation as of January 2025._

## 1. Design Objectives

The main requirements we aimed to satisfy in our design:

- Permissionless (i.e. no need for UNL approval to deploy a smart contract)
- Easy access to native features/primitives (to build on the XRPL's powerful building blocks)
- Easy learning for new developers (i.e. familiar design paradigms)
- Minimal impact to existing users/use-cases of the XRPL (especially with regard to payments and performance)
- Minimal impact to node/validator costs
- Able to auto-generate some sort of [ABI](https://www.alchemy.com/overviews/what-is-an-abi-of-a-smart-contract-examples-and-usage) from smart contract source code (to make tooling easier)

One of the great advantages of the XRPL is its human-readable transaction structure - unlike e.g. EVM transactions. In this design, we tried to keep that ethos of things being human-readable - such as keeping the ABI on-chain (even though that would increase storage).

### 1.1. General XRPL Programmability Vision

We envision programmability on the XRPL as the glue that seamlessly connects its powerful, native building blocks with the flexibility of custom on-chain business logic. This vision focuses on preserving what makes the XRPL special—its efficiency, reliability, and simplicity—while empowering builders to unlock new possibilities.

See [the blog post here](https://dev.to/ripplexdev/a-proposed-vision-for-xrp-ledger-programmability-1gdj) for more details.

## 2. Overview

This design for Smart Contracts combines the easy-to-learn overall design of EVM smart contracts (addresses with functions) with the familiarity of XRPL transactions. A Smart Contract lives on a [pseudo-account](https://github.com/XRPLF/XRPL-Standards/discussions/191) and is triggered via a new `ContractCall` transaction, which calls a specific function on the smart contract, with provided parameters. The Smart Contract can modify its own state data, or interact with other XRPL building blocks (including other smart contracts) via submitting its own XRPL transactions via its code.

The details of the WASM engine and the API will be in separate XLSes published later.

This proposal involves:

- Three new ledger entry types: `ContractSource`, `Contract`, and `ContractData`
- Six new transaction types: `ContractCreate`, `ContractCall`, `ContractModify`, `ContractDelete`, `ContractUserDelete`, and `ContractClawback`
- Two new RPC methods: `contract_info` and `event_history`
- One new RPC subscription: `eventEmitted`
- Modifications to the UNL-votable fee parameters (and the `FeeSettings` object that keeps track of that info)
<!--TODO: should probably work with Vito on this and think of this spec as essentially a more general SAV-->
- Three new serialized types: `STData`, `STDataType`, and `STJson`

### 2.1. Background: Pseudo-Accounts

A pseudo-account ([XLS-64d](https://github.com/XRPLF/XRPL-Standards/discussions/191)) is an XRPL account that is impossible for any person to have the keys for (it is cryptographically impossible to have those keys). It may be associated with other ledger entries.

Since it is not governed by any set of keys, it cannot be controlled by any user. Therefore, it may host smart contracts.

### 2.2. Background: Serialized Types

The XRPL encodes data into a set of serialized types (all of whose names begin with the letters `ST`, standing for "Serialized Type").

For example:

- The `"Account"` field is of type `STAccount` (which represents XRPL account IDs)
- The `"Sequence"` field is of type `STUInt32` (which represents an unsigned 32-bit integer)
- The `"Amount"` field is of type `STAmount` (which represents all amount types - XRP, IOUs, and MPTs)

### 2.3. Design Overview

- Smart contracts (henceforth referred to as just "contracts") are stored in pseudo-accounts
- `Contract` stores the contract info
- `ContractSource` is an implementation detail to make code storage more efficient
- `ContractData` stores any contract-specific data
- `ContractCreate` creates a new contract + pseudo-account, and allows the contract to do some setup work
- `ContractCall` is used to trigger the transaction and call one of its functions
- `ContractModify` allows the contract owner (or the contract itself) to modify a contract
- `ContractDelete` allows the contract owner (or the contract itself) to delete a contract
- `ContractUserDelete` allows a user of a smart contract to delete their data associated with a contract (and allows the contract to do any cleanup for that)
- `ContractClawback` allows a token issuer to claw back from a contract (and allows the contract to do any cleanup for that)
- There are some modifications to transaction common fields to support contract-submitted transactions
- The `contract_info` RPC fetches the ABI of a contract and any other relevant information
- The `event_history` RPC fetches the event emission history for a contract
- The `eventEmitted` subscription allows you to subscribe to "events" emitted from a contract
- All computation limitations and fees will be configurable by UNL vote (just like transaction fees and reserves are currently)

### 2.4. Overview of Smart Contract Capabilities

- Contract data storage using `STJson` for nested data structures
- Per-user contract data storage
- On-chain verified ABI with type information using `STDataType`
- Read-only access to ledger state (any ledger entries)
- Other changes to the ledger state are done via transactions that the pseudo-account "submits" from within contract code
- Contract-level "environment/class variable" parameters that don't require the source code to be changed
- May have an `init` function that runs on `ContractCreate` for any account setup

## 3. Ledger Entry: `ContractSource`

The objective of this object is to save space on-chain when deploying the exact same contract (i.e. if the same code is used by multiple contracts, the ledger only needs to store it once). This feature was heavily inspired by the existing Hooks `HookDefinition` object (see [this page](https://xrpl-hooks.readme.io/docs/reference-counting) for its documentation).

This object is essentially just an implementation detail to decrease storage costs, so that duplicate contracts don't need to have their source code copied on-chain. End-users won't have to worry about it. The core object in this design is the `Contract` object.

### 3.1. Fields

| Field Name           | Required? | JSON Type | Internal Type | Description                                                                                                                 |
| :------------------- | :-------- | :-------- | :------------ | :-------------------------------------------------------------------------------------------------------------------------- |
| `LedgerEntryType`    | ✔️        | `string`  | `UInt16`      | The ledger entry's type (`ContractSource`).                                                                                 |
| `ContractHash`       | ✔️        | `string`  | `Hash256`     | The hash of the contract's code.                                                                                            |
| `ContractCode`       | ✔️        | `string`  | `blob`        | The WebAssembly (WASM) bytecode for the contract.                                                                           |
| `InstanceParameters` |           | `array`   | `STArray`     | The parameters that are provided by a deployment of this contract (see structure below).                                    |
| `Functions`          | ✔️        | `array`   | `STArray`     | The functions that are included in this contract (see structure below).                                                     |
| `ReferenceCount`     | ✔️        | `string`  | `UInt64`      | The number of `Contract` objects that are based on this `ContractSource`. This object is deleted when that value goes to 0. |

#### 3.1.1. Object ID

hash of prefix + `ContractHash`

#### 3.1.2. `InstanceParameters` Structure

Each element in the `InstanceParameters` array is an object with:

```javascript
{
  "InstanceParameter": {
    "ParameterFlag": number,      // UInt32 - flags for the parameter
    "ParameterName": hex_string,  // Hex-encoded parameter name
    "ParameterType": STDataType   // Type descriptor (see Section 17)
  }
}
```

#### 3.1.3. `Functions` Structure

Each element in the `Functions` array is an object with:

```javascript
{
  "Function": {
    "FunctionName": hex_string,   // Hex-encoded function name
    "Parameters": [                // Array of parameter definitions
      {
        "Parameter": {
          "ParameterFlag": number,      // UInt32 - flags for the parameter
          "ParameterName": hex_string,  // Hex-encoded parameter name
          "ParameterType": STDataType   // Type descriptor
        }
      }
    ]
  }
}
```

- All parameter types must be valid XRPL `STypes`
- No more than 4 parameters allowed (for now - we can relax that restriction later)
- All parameters are required, no overloading allowed
- Function and parameter names are stored as hex-encoded strings for efficiency

#### 3.1.4. Parameter Flags

Possible flags for parameters:

- `tfSendAmount` (0x00000001) - if the type is an `STAmount`, then that amount will be sent to the contract from your funds
- `tfSendNFToken` (0x00000002) - if the type is a `Hash256`, then the `NFToken` with that ID will be sent to the contract from your holdings
- Others TBD - we have space for 32 possible flags

### 3.2. Object Deletion

The `ContractSource` object does not have an owner.

- The object is deleted if the `ReferenceCount` ever goes to 0
- This is enforced via invariant checks - no `ContractSource` object should exist with a `ReferenceCount` value of 0

### 3.3. Example Object

```javascript
{
  "LedgerEntryType": "ContractSource",
  "ContractHash": "E9366C4038209ED6092E61791A4043EE8131CD85C994F8891D034CD81FFC3249",
  "ContractCode": "0061736D01000000010E0260057F7F7F7F7F017F6000017F...",
  "Functions": [
    {
      "Function": {
        "FunctionName": "62617365", // hex for "base"
        "Parameters": [
          {
            "Parameter": {
              "ParameterFlag": 0,
              "ParameterName": "75696E7438", // hex for "uint8"
              "ParameterType": {
                "type": "UINT8"
              }
            }
          }
        ]
      }
    }
  ],
  "InstanceParameters": [
    {
      "InstanceParameter": {
        "ParameterFlag": 0,
        "ParameterName": "75696E7438", // hex for "uint8"
        "ParameterType": {
          "type": "UINT8"
        }
      }
    }
  ],
  "ReferenceCount": "1"
}
```

## 4. Ledger Entry: `Contract`

### 4.1. Fields

| Field Name                | Required? | JSON Type | Internal Type | Description                                                                                                   |
| :------------------------ | :-------- | :-------- | :------------ | :------------------------------------------------------------------------------------------------------------ |
| `LedgerEntryType`         | ✔️        | `string`  | `UInt16`      | The ledger entry's type (`Contract`).                                                                         |
| `ContractAccount`         | ✔️        | `string`  | `AccountID`   | The pseudo-account that hosts this contract.                                                                  |
| `Owner`                   | ✔️        | `string`  | `AccountID`   | The owner of the contract, which defaults to the account that deployed the contract.                          |
| `Flags`                   | ✔️        | `number`  | `UInt32`      | Flags that may be on this ledger entry.                                                                       |
| `Sequence`                | ✔️        | `number`  | `UInt32`      | The sequence number of the ContractCreate transaction that created this contract.                             |
| `ContractHash`            | ✔️        | `string`  | `Hash256`     | The hash of the contract's code (references a ContractSource).                                                |
| `InstanceParameterValues` |           | `array`   | `STArray`     | The values of the instance parameters for this contract deployment (see structure below).                     |
| `URI`                     |           | `string`  | `Blob`        | A URI that points to the source code of this contract, to make it easier for tooling to find the source code. |
| `OwnerNode`               | ✔️        | `string`  | `UInt64`      | A hint indicating which page of the contract account's owner directory links to this object.                  |

#### 4.1.1. Object ID

hash of prefix + `ContractHash` + `Sequence`

#### 4.1.2. `Flags`

- `lsfImmutable` (0x00010000) - the code can't be updated
- `lsfCodeImmutable` (0x00020000) - the code can't be updated, but the instance parameters can
- `lsfABIImmutable` (0x00040000) - the code can be updated, but the ABI cannot. This ensures backwards compatibility
- `lsfUndeletable` (0x00080000) - the contract can't be deleted

A `Contract` can have at most one of `lsfImmutable`, `lsfCodeImmutable`, and `lsfABIImmutable` enabled.

#### 4.1.3. `InstanceParameterValues` Structure

Each element in the `InstanceParameterValues` array is an object with:

```javascript
{
  "InstanceParameterValue": {
    "ParameterFlag": number,    // UInt32 - flags for the parameter
    "ParameterValue": STData    // Runtime-typed value (see Section 17)
  }
}
```

The instance parameter list must match the types specified in the corresponding `ContractSource` object's `InstanceParameters`.

### 4.2. Account Deletion

The `Contract` object is a [deletion blocker](https://xrpl.org/docs/concepts/accounts/deleting-accounts/#requirements).

### 4.3. Example Object

```javascript
{
  "LedgerEntryType": "Contract",
  "ContractAccount": "rNDM76PDyxgUBrhnw72UUYPQSwxxrZMkv6",
  "Owner": "rG1QQv2nh2gr7RCZ1P8YYcBUKCCN633jCn",
  "Flags": 0,
  "Sequence": 4,
  "ContractHash": "E9366C4038209ED6092E61791A4043EE8131CD85C994F8891D034CD81FFC3249",
  "InstanceParameterValues": [
    {
      "InstanceParameterValue": {
        "ParameterFlag": 0,
        "ParameterValue": {
          "type": "UINT8",
          "value": 1
        }
      }
    }
  ],
  "OwnerNode": "0"
}
```

## 5. Ledger Entry: `ContractData`

Data is serialized using the `STJson` serialization format (see Section 19).

### 5.1. Fields

| Field Name        | Required? | JSON Type | Internal Type | Description                                                                                                           |
| :---------------- | :-------- | :-------- | :------------ | :-------------------------------------------------------------------------------------------------------------------- |
| `LedgerEntryType` | ✔️        | `string`  | `UInt16`      | The ledger entry's type (`ContractData`).                                                                             |
| `Owner`           | ✔️        | `string`  | `AccountID`   | The account that hosts this data.                                                                                     |
| `ContractAccount` |           | `string`  | `AccountID`   | The contract that controls this data. This field is only needed for user-owned data (where the `Owner` is different). |
| `Data`            | ✔️        | `object`  | `STJson`      | The contract-defined contract data in JSON-like format.                                                               |

#### 5.1.1. Object ID

hash of prefix + `Owner` [+ `ContractAccount`]

### 5.2. Account Deletion

The `ContractData` object is a [deletion blocker](https://xrpl.org/docs/concepts/accounts/deleting-accounts/#requirements).

### 5.3. Reserves

Probably one reserve per 256 bytes

See [Reserves](#reserves)

### 5.4. Example Object

```javascript
{
  "LedgerEntryType": "ContractData",
  "Owner": "rWYkbWkCeg8dP6rXALnjgZSjjLyih5NXm",
  "ContractAccount": "rNDM76PDyxgUBrhnw72UUYPQSwxxrZMkv6",
  "Data": {
    "count": 3,
    "total": 12,
    "destination": "r3PDXzXky6gboMrwUrmSCiUyhzdrFyAbfu",
    "nested": {
      "subfield": "value"
    }
  }
}
```

## 6. Transaction: `ContractCreate`

This transaction creates a pseudo-account with the contract inside it.

This transaction will also trigger a special `init` function in the contract, if it exists - which allows smart contract devs to do their own setup work as needed.

### 6.1. Fields

| Field Name                | Required? | JSON Type | Internal Type | Description                                                                                   |
| :------------------------ | :-------- | :-------- | :------------ | :-------------------------------------------------------------------------------------------- |
| `TransactionType`         | ✔️        | `string`  | `UInt16`      | The transaction type (`ContractCreate`).                                                      |
| `Account`                 | ✔️        | `string`  | `AccountID`   | The account sending the transaction.                                                          |
| `ContractOwner`           |           | `string`  | `AccountID`   | The account that owns (controls) the contract. If not set, this is the same as the `Account`. |
| `Flags`                   |           | `number`  | `UInt32`      | Accepted bit-flags on this transaction.                                                       |
| `ContractCode`            |           | `string`  | `Blob`        | The WASM bytecode for the contract.                                                           |
| `ContractHash`            |           | `string`  | `Hash256`     | The hash of the WASM bytecode for the contract.                                               |
| `Functions`               |           | `array`   | `STArray`     | The functions that are included in this contract (required if ContractCode is provided).      |
| `InstanceParameters`      |           | `array`   | `STArray`     | The parameters that are provided by a deployment of this contract.                            |
| `InstanceParameterValues` |           | `array`   | `STArray`     | The values of the instance parameters that apply to this instance of the contract.            |

#### 6.1.1. `Flags`

- `tfImmutable` (0x00010000) - the code can't be changed and the instance parameters can't be updated either
- `tfCodeImmutable` (0x00020000) - the code can't be updated, but the instance parameters can
- `tfABIImmutable` (0x00040000) - the code can be updated, but the ABI cannot
- `tfUndeletable` (0x00080000) - the contract can't be deleted

A contract may have at most one of `tfImmutable`, `tfCodeImmutable`, and `tfABIImmutable` enabled.

#### 6.1.2. `ContractCode` and `ContractHash`

Exactly one of these two fields must be included.

`ContractCode` should be used if the code has not already been uploaded to the XRPL (i.e. there is no existing matching `ContractSource` object). This transaction will be more expensive.

`ContractHash` should be used if the code has already been uploaded to the XRPL. This transaction will be cheaper, since the code does not need to be re-uploaded.

If `ContractCode` is provided even if the code has already been uploaded, it will have the same outcome as if the `ContractHash` had been provided instead (albeit with a more expensive fee).

If `ContractCode` is provided, `InstanceParameters` and `Functions` must also be provided.

### 6.2. Fee

The fee is calculated as:

- Base transaction fee
- Plus one object reserve fee
- Plus fees per byte of code uploaded (if ContractCode is provided)
- Plus fees for running the `init` code (if it exists)

The fee calculation uses the `contractCreateFee()` function which takes the size of the bytecode as input.

### 6.3. Failure Conditions

- `ContractHash` is provided but there is no existing corresponding `ContractSource` ledger entry
- The `ContractCode` provided is invalid
- The ABI provided in `Functions` doesn't match the code
- `InstanceParameters` don't match what's in the existing `ContractSource` ledger entry
- Neither `ContractCode` nor `ContractHash` is present
- Both `ContractCode` and `ContractHash` are present

### 6.4. State Changes

If the transaction is successful:

- The pseudo-account that hosts the `Contract` is created
- The `Contract` object is created
- If the `ContractSource` object already exists, the `ReferenceCount` will be incremented
- If the `ContractSource` object does not already exist, it will be created with `ReferenceCount` set to 1
- The contract is linked to the pseudo-account's owner directory

### 6.5. Example Transaction

```javascript
{
  "TransactionType": "ContractCreate",
  "Account": "rG1QQv2nh2gr7RCZ1P8YYcBUKCCN633jCn",
  "Flags": 0,
  "ContractCode": "0061736D01000000010E0260057F7F7F7F7F017F6000017F...",
  "Functions": [
    {
      "Function": {
        "FunctionName": "62617365", // hex for "base"
        "Parameters": [
          {
            "Parameter": {
              "ParameterFlag": 0,
              "ParameterName": "75696E7438", // hex for "uint8"
              "ParameterType": {
                "type": "UINT8"
              }
            }
          }
        ]
      }
    }
  ],
  "InstanceParameterValues": [
    {
      "InstanceParameterValue": {
        "ParameterFlag": 0,
        "ParameterValue": {
          "type": "UINT8",
          "value": 1
        }
      }
    }
  ]
}
```

## 7. Transaction: `ContractCall`

This transaction triggers a specific function in a given contract, with the provided parameters.

### 7.1. Fields

| Field Name        | Required? | JSON Type | Internal Type | Description                                                                                              |
| :---------------- | :-------- | :-------- | :------------ | :------------------------------------------------------------------------------------------------------- |
| `TransactionType` | ✔️        | `string`  | `UInt16`      | The transaction type (`ContractCall`).                                                                   |
| `Account`         | ✔️        | `string`  | `AccountID`   | The account sending the transaction.                                                                     |
| `ContractAccount` | ✔️        | `string`  | `AccountID`   | The contract to call.                                                                                    |
| `FunctionName`    | ✔️        | `string`  | `Blob`        | The function on the contract to call (hex-encoded).                                                      |
| `Parameters`      |           | `array`   | `STArray`     | The parameters to provide to the contract's function. Must match the order and type of the on-chain ABI. |

### 7.2. Fee

The max number of instructions you're willing to run (gas-esque behavior)

### 7.3. Failure Conditions

- The `ContractAccount` doesn't exist or isn't a smart contract pseudo-account
- The function doesn't exist on the provided contract
- The parameters don't match the function's ABI

### 7.4. State Changes

If the transaction is successful, the WASM contract will be called. The WASM code will govern the state changes that are made.

### 7.5. Example Transaction

```javascript
{
  "TransactionType": "ContractCall",
  "Account": "rG1QQv2nh2gr7RCZ1P8YYcBUKCCN633jCn",
  "ContractAccount": "rNDM76PDyxgUBrhnw72UUYPQSwxxrZMkv6",
  "FunctionName": "63616C6C5F66756E6374696F6E", // hex for "call_function"
  "Parameters": [
    {
      "ParameterValue": {
        "ParameterFlag": 1, // tfSendAmount
        "ParameterValue": {
          "type": "AMOUNT",
          "value": "1000000" // 1 XRP
        }
      }
    },
    {
      "ParameterValue": {
        "ParameterFlag": 0,
        "ParameterValue": {
          "type": "AMOUNT",
          "value": {
            "currency": "USD",
            "issuer": "rJKnVATqzNsWa4jgnK5NyRKmK5s9QQWQYm",
            "value": "10"
          }
        }
      }
    },
    {
      "ParameterValue": {
        "ParameterFlag": 0,
        "ParameterValue": {
          "type": "ACCOUNT",
          "value": "rMJAmiEQW4XUehMixb9E8sMYqgsjKfB1yC"
        }
      }
    }
  ]
}
```

## 8. Transaction: `ContractModify`

This transaction modifies a contract's code, instance parameters, or ownership.

### 8.1. Fields

| Field Name                | Required? | JSON Type | Internal Type | Description                                                                                                                                        |
| :------------------------ | :-------- | :-------- | :------------ | :------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TransactionType`         | ✔️        | `string`  | `UInt16`      | The transaction type (`ContractModify`).                                                                                                           |
| `Account`                 | ✔️        | `string`  | `AccountID`   | The account sending the transaction.                                                                                                               |
| `ContractAccount`         |           | `string`  | `AccountID`   | The pseudo-account hosting the contract that is to be changed. This field is not needed if the pseudo-account is emitting this transaction itself. |
| `Owner`                   |           | `string`  | `AccountID`   | The new owner of the contract (to transfer ownership).                                                                                             |
| `Flags`                   |           | `number`  | `UInt32`      | Accepted bit-flags on this transaction.                                                                                                            |
| `ContractCode`            |           | `string`  | `Blob`        | The new WASM bytecode for the contract.                                                                                                            |
| `ContractHash`            |           | `string`  | `Hash256`     | The hash of the new WASM bytecode for the contract.                                                                                                |
| `Functions`               |           | `array`   | `STArray`     | The functions that are included in this contract (required if ContractCode is provided).                                                           |
| `InstanceParameters`      |           | `array`   | `STArray`     | The parameters that are provided by a deployment of this contract.                                                                                 |
| `InstanceParameterValues` |           | `array`   | `STArray`     | The values of the instance parameters that apply to this instance of the contract.                                                                 |

#### 8.1.1. `Flags`

Same as `ContractCreate` flags.

### 8.2. Fee

The fee is calculated similarly to `ContractCreate`:

- Base transaction fee
- Plus fees per byte of code uploaded (if ContractCode is provided)
- Plus fees for running any `init` code

### 8.3. Failure Conditions

- The `ContractAccount` doesn't exist or isn't a contract pseudo-account
- The `Account` isn't the contract owner (if `ContractAccount` is specified)
- If `ContractAccount` isn't specified, the `Account` isn't a contract pseudo-account
- The contract has an `lsfImmutable` flag
- The contract has `lsfCodeImmutable` enabled and `ContractCode` or `ContractHash` is provided
- The contract has `lsfABIImmutable` enabled and the ABI changes are not backwards-compatible
- Both `ContractCode` and `ContractHash` are present
- The new owner doesn't exist (if `Owner` is provided)
- The new owner is the same as the current account or the contract account itself

### 8.4. State Changes

If the transaction is successful:

- The `Contract` object is updated with new parameters/code/owner as specified
- If the code is changed:
  - The reference count of the old `ContractSource` is decremented
  - If the old `ContractSource` reference count reaches 0, it is deleted
  - If a new `ContractSource` doesn't exist, it is created with reference count 1
  - If a new `ContractSource` exists, its reference count is incremented
- If only instance parameters are changed, no `ContractSource` changes occur

### 8.5. Example Transaction

```javascript
{
  "TransactionType": "ContractModify",
  "Account": "rG1QQv2nh2gr7RCZ1P8YYcBUKCCN633jCn",
  "ContractAccount": "rNDM76PDyxgUBrhnw72UUYPQSwxxrZMkv6",
  "ContractHash": "E9366C4038209ED6092E61791A4043EE8131CD85C994F8891D034CD81FFC3249",
  "Functions": [
    {
      "Function": {
        "FunctionName": "62617365",
        "Parameters": [
          {
            "Parameter": {
              "ParameterFlag": 0,
              "ParameterName": "75696E7438",
              "ParameterType": {
                "type": "UINT8"
              }
            }
          }
        ]
      }
    }
  ],
  "InstanceParameterValues": [
    {
      "InstanceParameterValue": {
        "ParameterFlag": 0,
        "ParameterValue": {
          "type": "UINT8",
          "value": 1
        }
      }
    }
  ]
}
```

## 9. Transaction: `ContractDelete`

This transaction deletes a contract. Only the owner of the contract can do so.

### 9.1. Fields

| Field Name        | Required? | JSON Type | Internal Type | Description                                                                                                                                        |
| :---------------- | :-------- | :-------- | :------------ | :------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TransactionType` | ✔️        | `string`  | `UInt16`      | The transaction type (`ContractDelete`).                                                                                                           |
| `Account`         | ✔️        | `string`  | `AccountID`   | The account sending the transaction.                                                                                                               |
| `ContractAccount` |           | `string`  | `AccountID`   | The pseudo-account hosting the contract that is to be deleted. This field is not needed if the pseudo-account is emitting this transaction itself. |

### 9.2. Failure Conditions

- The `ContractAccount` doesn't exist or isn't a smart contract pseudo-account
- The `ContractAccount` holds deletion blocker objects (e.g. `Escrow` or `ContractData`)
- The account submitting the transaction is not the owner of the contract
- The contract has the `lsfUndeletable` flag set

### 9.3. State Changes

If the transaction is successful:

- The contract and its pseudo-account are deleted
- The reference count of the associated `ContractSource` is decremented
- If the `ContractSource` reference count reaches 0, it is deleted
- All objects that are still owned by the account (which are not deletion blockers) are deleted
- All remaining XRP in the account is returned to the owner

### 9.4. Example Transaction

```javascript
{
  "TransactionType": "ContractDelete",
  "Account": "rG1QQv2nh2gr7RCZ1P8YYcBUKCCN633jCn",
  "ContractAccount": "rNDM76PDyxgUBrhnw72UUYPQSwxxrZMkv6"
}
```

## 10. Transaction: `ContractUserDelete`

This transaction allows a user to delete their data associated with a contract. Only the user can submit this transaction (if the contract wants to modify user data, it can do that from the WASM code).

This transaction will also trigger a special `user_delete` function in the contract, if it exists - which allows smart contract devs to do their own cleanup work as needed.

### 10.1. Fields

| Field Name        | Required? | JSON Type | Internal Type | Description                                                                                                                                        |
| :---------------- | :-------- | :-------- | :------------ | :------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TransactionType` | ✔️        | `string`  | `UInt16`      | The transaction type (`ContractUserDelete`).                                                                                                       |
| `Account`         | ✔️        | `string`  | `AccountID`   | The account sending the transaction (the user).                                                                                                    |
| `ContractAccount` |           | `string`  | `AccountID`   | The pseudo-account hosting the contract that is to be changed. This field is not needed if the pseudo-account is emitting this transaction itself. |

### 10.2. Failure Conditions

- The `ContractAccount` doesn't exist or isn't a smart contract pseudo-account
- The `Account` does not have any `ContractData` object for the contract in `ContractAccount`

### 10.3. State Changes

If the transaction is successful:

- The user's `ContractData` object associated with the `ContractAccount` will be deleted
- The `user_delete` function on the contract will be run to perform any cleanup work, if it exists

## 11. Transaction: `ContractClawback`

This transaction allows issuers to claw back tokens from a contract, while also allowing smart contract devs to perform any cleanup they need to based on this clawback result.

This transaction will trigger a special `clawback` function in the contract, if it exists - which allows smart contract devs to do their own cleanup work as needed.

### 11.1. Fields

| Field Name        | Required? | JSON Type | Internal Type | Description                                                    |
| :---------------- | :-------- | :-------- | :------------ | :------------------------------------------------------------- |
| `TransactionType` | ✔️        | `string`  | `UInt16`      | The transaction type (`ContractClawback`).                     |
| `Account`         | ✔️        | `string`  | `AccountID`   | The account sending the transaction (the issuer of the token). |
| `ContractAccount` |           | `string`  | `AccountID`   | The pseudo-account hosting the contract that is to be changed. |
| `Amount`          | ✔️        | `object`  | `Amount`      | The amount to claw back from the contract.                     |

### 11.2. Failure Conditions

- The `ContractAccount` doesn't exist or isn't a smart contract pseudo-account
- `Amount` is invalid in some way (e.g. is negative, token doesn't exist, is XRP)
- The `Account` isn't the issuer of the token specified in `Amount`
- The `ContractAccount` doesn't hold the token specified in `Amount`
- The `ContractAccount` holds the token specified in `Amount`, but holds less than the amount specified

### 11.3. State Changes

If the transaction is successful:

- The balance of the `ContractAccount`'s token is decreased by `Amount`

## 12. Transaction Common Fields

This standard doesn't add any new field to the [transaction common fields](https://xrpl.org/docs/references/protocol/transactions/common-fields/), but it does add another global transaction flag and add another metadata field.

### 12.1. `Flags`

| Flag Name                | Value        |
| :----------------------- | :----------- |
| `tfContractSubmittedTxn` | `0x20000000` |

This flag should only be used if a transaction is submitted from a smart contract. This signifies that the transaction shouldn't be signed. Any transaction that is submitted normally that includes this flag should be rejected.

Contract-submitted transactions will be processed in a method very similar to [Batch inner transactions](https://github.com/XRPLF/XRPL-Standards/tree/master/XLS-0056d-batch) - i.e. executed within the `ContractCall` processing, rather than as a separate independent transaction. This allows the smart contract code to take actions based on whether the transaction was successful.

### 12.2. Metadata

Every contract-submitted transaction will contain an extra metadata field, `ParentContractCallId`, containing the hash of the `ContractCall` transaction that triggered its submission.

## 13. RPC: `contract_info`

This RPC fetches info about a deployed contract.

### 13.1. Request Fields

| Field Name         | Required? | JSON Type | Description                                             |
| :----------------- | :-------- | :-------- | :------------------------------------------------------ |
| `contract_account` | ✔️        | `string`  | The pseudo-account hosting the contract.                |
| `function`         |           | `string`  | The function to specifically get information for.       |
| `user_account`     |           | `string`  | An account to specifically get the contract's data for. |

### 13.2. Response Fields

| Field Name         | Always Present?                              | JSON Type | Description                                                                  |
| :----------------- | :------------------------------------------- | :-------- | :--------------------------------------------------------------------------- |
| `contract_account` | ✔️                                           | `string`  | The pseudo-account hosting the contract.                                     |
| `code`             | ✔️                                           | `string`  | The WASM bytecode for the contract.                                          |
| `account_info`     | ✔️                                           | `object`  | The `account_info` output of the `contract_account`.                         |
| `functions`        | ✔️                                           | `array`   | The functions in the smart contract and their parameters.                    |
| `source_code_uri`  |                                              | `string`  | The URI pointing to the source code of the contract (if it exists on-chain). |
| `contract_data`    |                                              | `object`  | The contract's stored data.                                                  |
| `user_data`        | If `user_account` is included in the request | `object`  | The contract's stored data pertaining to that user.                          |

#### 13.2.1. `functions`

Each object in the array will contain the following fields:

| Field Name   | Always Present? | JSON Type | Description                                                                                                                                       |
| :----------- | :-------------- | :-------- | :------------------------------------------------------------------------------------------------------------------------------------------------ |
| `name`       | ✔️              | `string`  | The name of the function (decoded from hex).                                                                                                      |
| `parameters` | ✔️              | `array`   | A list of the parameters accepted for the function. This will be an empty list if the function takes no parameters.                               |
| `fees`       | ✔️              | `string`  | The amount, in XRP, that you would likely have to pay to execute this function. _TODO: not sure how doable this is, but it'd be nice if possible_ |

## 14. RPC Subscription: `eventEmitted`

Subscribe to events emitted from a contract.

### 14.1. Request Fields

| Field Name         | Required? | JSON Type | Description                                                                                                                  |
| :----------------- | :-------- | :-------- | :--------------------------------------------------------------------------------------------------------------------------- |
| `contract_account` | ✔️        | `string`  | The pseudo-account hosting the contract.                                                                                     |
| `events`           |           | `array`   | The event types to subscribe to, as an `array` of `string`s. If omitted, all events from the contract will be subscribed to. |

### 14.2. Response Fields

| Field Name         | Always Present? | JSON Type | Description                                                         |
| :----------------- | :-------------- | :-------- | :------------------------------------------------------------------ |
| `contract_account` | ✔️              | `string`  | The pseudo-account hosting the contract.                            |
| `events`           | ✔️              | `array`   | The events that were emitted from the contract, as explained below. |
| `hash`             | ✔️              | `string`  | The hash of the transaction that triggered the event.               |
| `ledger_index`     | ✔️              | `number`  | The ledger index in which the event was triggered.                  |

#### 14.2.1. `events`

Each object in the `events` array will contain the following fields:

| Field Name | Always Present? | JSON Type | Description                              |
| :--------- | :-------------- | :-------- | :--------------------------------------- |
| `name`     | ✔️              | `string`  | The name of the event.                   |
| `data`     | ✔️              | `object`  | The data emitted as a part of the event. |

The rest of the fields in this object will be dev-defined fields from the emitted event.

## 15. RPC: `event_history`

Fetch a list of historical events emitted from a given contract account.

### 15.1. Request Fields

| Field Name         | Required? | JSON Type     | Description                                                                                                                                                                                                                                                                                                                        |
| :----------------- | :-------- | :------------ | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `contract_account` | ✔️        | `string`      | The pseudo-account hosting the contract.                                                                                                                                                                                                                                                                                           |
| `events`           |           | `array`       | The event types to retrieve, as an `array` of `string`s. If omitted, all events from the contract will be retrieved.                                                                                                                                                                                                               |
| `ledger_index_min` |           | `number`      | Use to specify the earliest ledger to include events from. A value of `-1` instructs the server to use the earliest validated ledger version available.                                                                                                                                                                            |
| `ledger_index_max` |           | `number`      | Use to specify the most recent ledger to include events from. A value of `-1` instructs the server to use the most recent validated ledger version available.                                                                                                                                                                      |
| `ledger_index`     |           | `LedgerIndex` | Use to look for events from a single ledger only.                                                                                                                                                                                                                                                                                  |
| `ledger_hash`      |           | `string`      | Use to look for events from a single ledger only.                                                                                                                                                                                                                                                                                  |
| `binary`           |           | `boolean`     | Defaults to `false`. If set to `true`, returns events as hex strings instead of JSON.                                                                                                                                                                                                                                              |
| `limit`            |           | `number`      | Default varies. Limit the number of events to retrieve. The server is not required to honor this value.                                                                                                                                                                                                                            |
| `marker`           |           | `any`         | Value from a previous paginated response. Resume retrieving data where that response left off. This value is stable even if there is a change in the server's range of available ledgers. See [here](https://xrpl.org/docs/references/http-websocket-apis/api-conventions/markers-and-pagination) for details on how markers work. |
| `transactions`     |           | `boolean`     | Defaults to `false`. If set to `true`, returns the whole transaction in addition to the event. If set to false, returns only the transaction hash.                                                                                                                                                                                 |

### 15.2. Response Fields

| Field Name         | Always Present? | JSON Type | Description                                                                                                               |
| :----------------- | :-------------- | :-------- | :------------------------------------------------------------------------------------------------------------------------ |
| `contract_account` | ✔️              | `string`  | The pseudo-account hosting the contract.                                                                                  |
| `events`           | ✔️              | `array`   | The events that were emitted from the contract.                                                                           |
| `ledger_index_min` |                 | `number`  | The ledger index of the earliest ledger actually searched for events.                                                     |
| `ledger_index_max` |                 | `number`  | The ledger index of the most recent ledger actually searched for events.                                                  |
| `limit`            |                 | `number`  | The limit value used in the request. (This may differ from the actual limit value enforced by the server.)                |
| `marker`           |                 | `any`     | Server-defined value indicating the response is paginated. Pass this to the next call to resume where this call left off. |

#### 15.2.1. `events`

Each object in the `events` array will contain the following fields:

| Field Name  | Always Present? | JSON Type | Description                                                                                                                                                    |
| :---------- | :-------------- | :-------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`      | ✔️              | `string`  | The name of the event.                                                                                                                                         |
| `data`      |                 | `object`  | The data emitted as a part of the event, in JSON. Included if `binary` is set to `false`.                                                                      |
| `data_blob` |                 | `string`  | The data emitted as a part of the event, in binary. Included if `binary` is set to `true`.                                                                     |
| `tx_json`   |                 | `object`  | The transaction that triggered the event, in JSON. Included if `transactions` is set to `true` and `binary` is set to `false`.                                 |
| `tx_blob`   |                 | `string`  | The transaction that triggered the event, in binary. Included if `transactions` is set to `true` and `binary` is set to `true`.                                |
| `tx_hash`   |                 | `string`  | The transaction that triggered the event, in binary. Included if `transactions` is set to `false`.                                                             |
| `validated` |                 | `boolean` | Whether or not the transaction that contains this event is included in a validated ledger. Any transaction not yet in a validated ledger is subject to change. |

### 15.3. Implementation Details

This RPC will use the account transactions database. It will iterate through all `ContractCall` transactions sent to the provided `contract_account` and filter out the events based on the other provided parameters.

## 16. UNL-Votable Parameters

- Number of drops per instruction run (in some unit that allows this to be <1)
- Maximum number of instructions allowed in a single transaction
  - Maybe separate values for contracts vs subroutines (and maybe separate values for each subroutine)
- Memory limits
- Memory costs

Initial values will be high fees/low maxes, but via the standard UNL voting process for fees, this can be adjusted over time as needed.

## 17. Serialized Type: `STData`

`STData` is a runtime-typed value container that can hold any XRPL serialized type. It stores both the type information and the value itself.

### 17.1. Structure

The serialized format consists of:

- 16-bit type identifier (SerializedTypeID)
- The serialized value of that type

### 17.2. JSON Representation

```javascript
{
  "type": "UINT8",     // String representation of the type
  "value": 42          // The actual value (format depends on type)
}
```

### 17.3. Supported Types

- `UINT8`, `UINT16`, `UINT32`, `UINT64`
- `UINT128`, `UINT160`, `UINT192`, `UINT256`
- `VL` (variable length blob)
- `ACCOUNT` (AccountID)
- `AMOUNT` (STAmount - XRP or token amounts)
- `ISSUE` (STIssue - currency + issuer)
- `CURRENCY` (STCurrency)
- `NUMBER` (STNumber - floating point)

### 17.4. Example Usage

```javascript
// As a parameter value in ContractCall
{
  "ParameterFlag": 0,
  "ParameterValue": {
    "type": "UINT32",
    "value": 12345
  }
}

// As an Amount
{
  "ParameterFlag": 1,  // tfSendAmount
  "ParameterValue": {
    "type": "AMOUNT",
    "value": "1000000"  // 1 XRP
  }
}
```

## 18. Serialized Type: `STDataType`

`STDataType` represents just the type information without a value. Used in function and parameter definitions.

### 18.1. Structure

The serialized format consists of:

- 16-bit type identifier (SerializedTypeID)

### 18.2. JSON Representation

```javascript
{
  "type": "UINT8"     // String representation of the type
}
```

### 18.3. Example Usage

```javascript
// In function parameter definition
{
  "Parameter": {
    "ParameterFlag": 0,
    "ParameterName": "75696E7438",  // hex for "uint8"
    "ParameterType": {
      "type": "UINT8"
    }
  }
}
```

## 19. Serialized Type: `STJson`

`STJson` provides JSON-like nested data structures for contract data storage.

### 19.1. Features

- Supports objects (key-value maps) and arrays
- Maximum nesting depth of 1 (one level of nesting allowed)
- Values are stored as shared pointers to any STBase type
- Uses variable-length encoding with type markers

### 19.2. Structure

The serialized format consists of:

- VL length prefix
- 1-byte type marker (0x01 for array, 0x02 for object)
- For arrays: VL-encoded values (each with their own type marker)
- For objects: VL-encoded key-value pairs

### 19.3. JSON Representation

```javascript
// Object example
{
  "count": 3,
  "total": 12,
  "destination": "r3PDXzXky6gboMrwUrmSCiUyhzdrFyAbfu",
  "nested": {
    "subfield": "value"  // One level of nesting allowed
  }
}

// Array example
[1, "hello", {"key": "value"}]
```

### 19.4. Limitations

- Maximum nesting depth: 1
- Cannot nest STJson within STJson beyond one level
- Keys must be strings
- Values can be any STBase type (including nested STJson up to depth limit)

### 19.5. Example Usage in ContractData

```javascript
{
  "LedgerEntryType": "ContractData",
  "Owner": "rWYkbWkCeg8dP6rXALnjgZSjjLyih5NXm",
  "Data": {
    "userBalance": 1000,
    "lastUpdate": 1234567890,
    "settings": {
      "autoRenew": true,
      "threshold": 100
    }
  }
}
```

## 20. Examples

### 20.1. Sample Code

_Note: this is entirely made up, and is only intended to provide a rough idea of the concepts described in this document. The exact syntax is heavily subject to change (during both design iterations and in the implementation phase, as more details are figured out). For example, the final version will likely be in Rust._

```javascript
// This sample function pays the destination some funds and transfers some data to the destination
function transfer(UInt16 number, STAmountSend amount, AccountID destination)
{
  const { balance } = getUserData(this, this.caller)
  if (!balance || balance < number)
    reject("Not enough balance") // error message included in metadata
  const { balance: destBalance } = getUserData(this, destination)
  if (!destBalance)
    return "Destination hasn't authorized data" // failure
  setUserData(this, this.caller, { balance: balance - number })
  setUserData(this, destination, { balance: balance + number })
  submitTransaction({
    TransactionType: "Payment",
    Destination: destination,
    Amount: amount
  })
  console.log("Transfer finished") // This is printed in the rippled debug.log (for testing purposes)
  emitEvent("transfer", {
    account: this.caller,
    destination: destination,
    number: number,
    amount: amount
  }) // this is included in the metadata - there can be limits on how much data may be included
  return 0 // success
}
```

- Submitting transactions is like Hooks - the pseudo-account essentially "sends" transactions to do all of its on-chain modifications (other than contract-specific state)
- There's a special `init()` function that allows you to do any initial account setup (instead of needing to do it in a separate function call)
- `emitEvent` is used to emit an event that is stored in the metadata

## 21. Invariants

- No `ContractSource` object should have a `ReferenceCount` of 0
- Every `Contract` object should have an existing corresponding `ContractSource` object
- A `Contract` cannot have more than one of `lsfImmutable`, `lsfCodeImmutable`, and `lsfABIImmutable` enabled
- `STJson` objects cannot have nesting depth greater than 1

## 22. Security

### 22.1. Pseudo-Account Account-Level Permissions

These settings will all be enabled by default on a pseudo-account:

- Disable master key
- Enable `DepositAuth`

These functions will all be disallowed from pseudo-accounts:

- `SignerListSet`
- `SetRegularKey`
- Enable master key
- Disable `DepositAuth`
- `AccountPermissionsSet`

This prevents the contract account from receiving any funds directly (i.e. outside of the `ContractCall` transaction) and prevents any other account (malicious or otherwise) from submitting transactions directly from the contract account. This ensures that the contract account's logic is entirely governed by the code, and nothing else.

### 22.2. Scam Contracts

Any amount you send to a contract can be rug-pulled. This is the same level of risk as any other scam on the XRPL right now - such as buying a rug-pulled token.

### 22.3. Fee-Scavenging/Dust Attacks

Don't think this is possible here.

### 22.4. Re-entrancy Attacks

These will need to be guarded against in this design. One option for this is to disallow any recursion in calls - i.e. disallow `ContractCall` transactions from a contract account to itself. Other options are being investigated.

# Open Questions

- Pre-load all the contract instance params when opening the VM so it's not expensive to fetch them?
- How should a contract handle if someone sends a token they don't have a trustline for?
- Readonly contract functions like EVM? Might be for v2, or might be unnecessary given the human-readability of the `STJson` type
- Should `ContractData` objects be stored in a separate directory structure (e.g. a new one, not the standard owner directory)?
- What should happen if `user_delete` or `clawback` fails or crashes in some way?
  - If it crashes due to running out of `ComputationAmount` the transaction should probably fail, if it crashes/fails for anything else (that is a result of the WASM code) the transaction should probably succeed

## Reserves

The biggest remaining question (from the core ledger design perspective) is how to handle object reserves for any contract-specific data (reserves for all existing ledger entries can be handled as they are now).

The most naive option is for the contract account to hold all necessary funds for the reserves. However, this is quite the burden on the contract account (and therefore the deployer of the contract). Ideally, there should be some way to put some of the burden on the contract users for the parts of the data that they use.

One example to illustrate the difference: an ERC-20 contract holds all of its data (e.g. holders and the amount they hold) in the contract itself. However, on the XRPL, an MPT issuer does not need to cover reserves for all of its holders and their data - only the reserves for the issuance data. The EVM doesn't have the concept of reserves (it's essentially amortized in the transaction fees), this concern doesn't apply to those chains.

### Account Reserve

The account reserve is essentially covered by the non-refundable `ContractCreate` transaction fee. This also covers the reserve for the `Contract`/`ContractSource` ledger entry, if needed.

### Object Reserve

The current implementation leans towards:

**User data and reserves are handled by the user**

- Inspired by Move
- A "user data" object for contracts to store data for a particular user
- Hash is `Contract ID` + `Account`, only one object stores all of a user's data
- N bytes per reserve charged (perhaps N=256), with some sort of limit towards the max amount of reserves that can be charged
- Transaction to delete user data (to recover reserves) - contracts need to support this
- Object is a deletion blocker (for the contract and the user)

The authors lean towards this approach, as it feels the most XRPL-y, but supporting data deletion becomes complicated, because otherwise contract developers can lock up users' reserves without them being able to free that reserve easily.

# Appendix

## Appendix A: FAQ

### A.1: How does this compare to Hooks?

The main similarities:

- The smart contract API is mostly the same. The biggest changes are renaming functions (and a couple of extra methods added)
- Smart contracts interact with XRPL primitives by submitting/emitting transactions
- A "contract definition" object is used so that the ledger doesn't need to store the same data repeatedly
- A WASM VM is used to process the smart contract code

The main differences:

- Smart contracts are installed on pseudo-accounts instead of user accounts
- Instead of being triggered by transactions, there is a special `ContractCall` transaction to trigger the smart contract
- Smart contract data is stored using the flexible `STJson` type
- Users handle the reserves for their own data, instead of it all being handled by the smart contract account
- Smart contracts can emit events that developers can subscribe to
- Tools can auto-generate ABIs from source code (and ABIs are stored on-chain)
- Function and parameter names are hex-encoded for efficiency

### A.2: How does this compare to EVM?

The main similarities:

- Smart contracts are installed at their own addresses
- Smart contracts are organized by functions, and they are called by transactions that encode the function to call and its parameters
- Smart contracts can emit events that developers can subscribe to

The main differences:

- Since the XRPL has native features/primitives (unlike EVM chains), transaction emission allows smart contracts to interact with those primitives
- Additional complexities due to the XRPL's reserve system
- A "contract definition" object is used so that the ledger doesn't need to store the same data repeatedly (for example, Uniswap and ERC-20 contracts are repeatedly deployed onto EVM chains, adding a lot of data bloat)
- The VM used is WASM, not EVM
- ABIs are stored on-chain
- Uses flexible runtime-typed parameters with `STData`

### A.3: How can I implement account logic (like in Hooks) with this form of smart contracts?

Use something akin to Ethereum's Account Abstraction design ([ERC-4337](https://www.erc4337.io/)).

Might involve [XLS-75d](https://github.com/XRPLF/XRPL-Standards/discussions/218) (Delegating Account Permissions).

We're also investigating whether additional Smart Features can help with this problem.

### A.4: Will I be able to transfer (or copy/paste) EVM/SolVM/MoveVM/etc. bytecode to the XRPL?

No (well, not without a special tool that will do the conversion for you).

### A.5: Will I be able to write smart contracts in Solidity?

Solidity will not be prioritized by our team right now, but there are Solidity-to-WASM compilers that someone could use to make this possible.

Note that the syntax would likely not be the exact same as in the EVM. In addition, the conceptual meanings may not map either - for example, addresses are different between the EVM and the XRPL.

### A.6: Will I be able to implement something akin to ERC-20 with an XRPL smart contract?

Yes, but it will be more expensive and less efficient than the existing token standards (IOUs and MPTs) and won't be integrated into the DEX or other parts of the XRPL.

An alternative strategy would be to create a smart contract that essentially acts as a wrapper to the XRPL's native functionalities (e.g. a `mint` function that just issues a token via a `Payment` transaction).

### A.7. How will fees be handled for contract-submitted transactions?

It'll be included in the fees paid for the contract call.

### A.8. What happens if a smart contract pseudo-account's funds are clawed back, or are locked/frozen?

That's for the smart contract author to deal with, just like in the EVM world.

### A.9: What languages can/will be supported?

Any language that can compile to WASM can be supported. We will likely start with Rust.

### A.10: Can a smart contract execute a multi-account Batch transaction with another account?

Yes, if the smart contract account submits the final transaction. Constructing a multi-account Batch transaction between two smart contracts will not be possible as a part of this spec. Support for that could be added with a separate design.

### A.11: Can I use [Smart Escrows](https://docs.google.com/document/u/0/d/1upiaBSHF8pTeHvdHcJ0BwPxwFK_znpsSJr1PToZY0Do/edit) with Smart Contracts?

Yes, a smart contract can emit an `EscrowCreate` transaction that has a `FinishFunction`.

### A.12: Will this design support read-only functions like EVM?

Not in the initial version/design, to keep it simple. This could be added in the future, though.

### A.13: How do I get the transaction history of a Contract Account?

Use the existing `account_tx` RPC.

### A.14: How does this design prevent abuse/infinite loops from eating up rippled resources, while allowing for sufficient compute for smart contract developers?

The `UNL-Votable Parameters` section addresses this. The UNL can adjust the parameters based on the needs and limitations of the network, to ensure that developers have enough computing resources for their needs, while ensuring that they cannot overrun the network with their contracts.

### A.15: Why store the ABI on-chain instead of in an Etherscan-like system?

Having the data on-chain removes the need for a centralized party to maintain this data - all the data to interact with a contract is available on-chain. This also means that it's much easier (and therefore faster) for `rippled` to determine if the data passed into a contract function is valid, instead of needing to open up the WASM engine for that.

The tradeoff is that this means a contract will take up more space on-chain, and we need new STypes to store that information properly.
