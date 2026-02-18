<pre>
  title: Binary Codec RPCs
  description: Standard admin RPCs to encode and decode XRPL objects between JSON and binary forms
  author: Mayukha Vadari (@mvadari), Alex Kremer (@godexsoft)
  category: System
  status: Draft
  proposal-from: https://github.com/XRPLF/rippled/issues/6139
  created: 2026-02-18
</pre>

## 1. Abstract

This document specifies two administrative XRP Ledger RPC methods, `encode` and `decode`, that convert between canonical XRPL JSON objects and the binary wire format used by rippled. Both methods use the same field and type definitions exposed by the existing `server_definitions` RPC (mirroring `definitions.json` from `xrpl.js`), including in‑progress serialization types.

The `encode` method serializes an arbitrary collection of XRPL fields (a generic STObject) from JSON to canonical binary hex; the `decode` method parses a binary blob into JSON using the active definitions. A `strict` flag optionally applies transaction or ledger‑entry format validation when the object includes `TransactionType` or `LedgerEntryType`, while unknown field names and unknown field IDs are always rejected.

## 2. Motivation

Client libraries already implement encoding and decoding using shared definitions (for example `xrpl.js`), but they may lag behind network deployments (for example devnets, sidechains, or experimental features with new STypes). Tooling and testing frameworks also need a canonical source of truth for how the server currently interprets definitions, including in‑progress ones.

Today there is no standard, server‑side RPC that encodes arbitrary STObjects using the server’s active definitions and decodes arbitrary binary blobs back to JSON using the same definitions. Defining `encode` and `decode` addresses this by providing a canonical server‑side codec that works for all active and experimental STypes and enables robust tooling, fuzzing, and test harnesses directly against a running server.

## 3. Specification

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL are to be interpreted as described in RFC 2119 and RFC 8174.

### 3.1. Definitions and Scope

- **STObject**: The generic XRPL serialized object type (a collection of fields).
- **Definitions**: The effective set of field and type definitions that rippled exposes via the `server_definitions` RPC (corresponding to `FIELDS`, `TYPES`, `TRANSACTION_TYPES`, and `LEDGER_ENTRY_TYPES` in `definitions.json`).
- **Field ID**: The numeric identifier for a field, derived from its `(type_code, nth)` pair.
- **Transaction candidate**: A JSON object that includes a `TransactionType` field whose value resolves to a known transaction type.
- **Ledger entry candidate**: A JSON object that includes a `LedgerEntryType` field whose value resolves to a known ledger entry type.

These RPCs MUST NOT modify ledger state or participate in consensus. Implementations MUST use the server’s own active definitions when encoding and decoding, including any locally configured in‑progress STypes on devnets or sidechains.

#### 3.1.1. Binary encoding rules

`encode` MUST:

- Use the same field ordering and wire format as rippled uses for STObjects (canonical sort by `(type_code, nth)`, field ID prefixes, and length prefixes where applicable).
- For fields marked with `isSerialized == false` in `FIELDS`, MUST NOT include them in the binary; they MAY be accepted in input JSON but MUST be dropped from the encoded payload and from any normalized JSON returned.

`decode` MUST parse the binary according to the same field ID and type definitions, reconstructing a JSON object keyed by field names.

### 3.2. RPC: `encode`

`encode` serializes a JSON object representing an arbitrary collection of XRPL fields (a generic STObject) into a single canonical binary blob, returned as a hexadecimal string.

#### 3.2.1. Request Fields

For WebSocket and command-style APIs, the request object MUST include the following fields:

| Field Name | Required? | JSON Type | Description                                                                                                                                    |
| ---------- | --------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| json       | Yes       | object    | Arbitrary collection of XRPL fields to encode. Keys MUST be XRPL field names present in `FIELDS`, and values MUST conform to the field’s type. |
| strict     | No        | boolean   | If `true`, apply additional validation when `TransactionType` or `LedgerEntryType` is present (see below). Defaults to `false`.                |

For JSON-RPC, these fields MUST appear inside the single params object and the top-level `method` MUST be `"encode"`.

Additional rules:

- All keys in `json` MUST exist in `FIELDS`. Any unknown field name MUST cause the server to reject the request.
- If `strict` is `false`, only field-level validation is applied; the server MUST serialize the STObject even if it would not be a valid transaction or ledger entry.
- If `strict` is `true`:
  - If `json` contains `TransactionType` that resolves to a known transaction type, the object MUST be validated as a transaction candidate (required fields present, illegal fields absent, basic invariants). On failure, the server MUST return an error and MUST NOT return a `binary` result.
  - Else if `json` contains `LedgerEntryType` that resolves to a known ledger entry type, the object MUST be validated as a ledger entry candidate with similar rules.
  - Else, no additional structural validation is applied beyond field-level checks.
  - If both `TransactionType` and `LedgerEntryType` are present, the request MUST be rejected as ambiguous.

#### 3.2.2. Response Fields

On success, the `result` object MUST include:

| Field Name | Always Present? | JSON Type | Description                                                                                                                                                                           |
| ---------- | --------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| binary     | Yes             | string    | Canonical binary encoding of the STObject, as a hex string containing only `[0-9a-fA-F]` and of even length.                                                                          |
| json       | No              | object    | Normalized JSON that was actually encoded. MUST NOT contain any fields with `isSerialized == false` and MUST correspond to decoding the returned `binary` under the same definitions. |

#### 3.2.3. Failure Conditions

`encode` MUST return an error at least for:

1. **Unknown field name**: any key in `json` not present in `FIELDS` (for example, an `unknownField` error).
2. **Type mismatch**: a value that cannot be parsed as the declared type.
3. **Ambiguous object kind** when `strict: true` and both `TransactionType` and `LedgerEntryType` are present.
4. **Invalid transaction or ledger entry format** when `strict: true` and the candidate fails its schema.
5. **Internal serialization errors**.

#### 3.2.4. Example Request

```json
{
  "command": "encode",
  "json": {
    "TransactionType": "Payment",
    "Account": "rEXAMPLESOURCE",
    "Destination": "rEXAMPLEDEST"
  },
  "strict": true
}
```

#### 3.2.5. Example Response

```json
{
  "status": "success",
  "binary": "12000022800000002400000001...",
  "json": {
    "TransactionType": "Payment",
    "Account": "rEXAMPLESOURCE",
    "Destination": "rEXAMPLEDEST"
  }
}
```

### 3.3. RPC: `decode`

`decode` parses an XRPL binary blob representing an STObject and returns a JSON object whose fields and types are derived from the server’s active definitions.

#### 3.3.1. Request Fields

For WebSocket and command-style APIs, the request object MUST include the following fields:

| Field Name | Required? | JSON Type | Description                                                                                                                                      |
| ---------- | --------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| command    | Yes       | string    | Must be `"decode"`.                                                                                                                              |
| binary     | Yes       | string    | Hex string representing the STObject; MUST be well-formed hex of even length.                                                                    |
| strict     | No        | boolean   | If `true`, apply additional validation when the decoded object contains `TransactionType` or `LedgerEntryType` (see below). Defaults to `false`. |

For JSON-RPC, these fields MUST appear inside the single params object and the top-level `method` MUST be `"decode"`.

Additional rules:

- The `binary` string MUST be valid hex and have even length.
- If `strict` is `false`, any syntactically valid STObject whose field IDs and values parse correctly MUST be decoded with no extra structural validation.
- If `strict` is `true` and the decoded object contains `TransactionType` or `LedgerEntryType`, it MUST be validated as in `encode`; ambiguous presence of both MUST be rejected.

#### 3.3.2. Response Fields

On success, the `result` object MUST include:

| Field Name | Always Present? | JSON Type | Description                                    |
| ---------- | --------------- | --------- | ---------------------------------------------- |
| status     | Yes             | string    | `"success"` if the request succeeded.          |
| json       | Yes             | object    | Decoded JSON object keyed by XRPL field names. |

Implementations MAY also include `binary` (canonical hex) or `warnings`, but clients MUST NOT rely on these.

#### 3.3.3. Failure Conditions

`decode` MUST return an error at least for:

1. **Invalid hex**: non-hex characters or odd length.
2. **Unknown field ID**: field ID not present in `FIELDS` under current definitions.
3. **Type decoding errors**: binary representation for a field cannot be parsed as the declared type.
4. **Ambiguous object kind** when `strict: true` and both `TransactionType` and `LedgerEntryType` are present.
5. **Invalid transaction or ledger entry format** when `strict: true`.
6. **Malformed STObject**: truncated field, trailing bytes, or similar framing errors.

#### 3.3.4. Example Request

```json
{
  "command": "decode",
  "binary": "12000022800000002400000001...",
  "strict": true
}
```

#### 3.3.5. Example Response

```json
{
  "status": "success",
  "json": {
    "TransactionType": "Payment",
    "Account": "rEXAMPLESOURCE",
    "Destination": "rEXAMPLEDEST"
  }
}
```

## 4. Rationale

Two RPCs (`encode` and `decode`) are used instead of a single `codec` method with an `action` parameter to keep each RPC focused on one direction and to simplify schemas, tooling, and documentation. No `object_type` parameter is defined; instead, objects are treated as generic STObjects, and transaction or ledger‑entry semantics are inferred only from `TransactionType` and `LedgerEntryType` when present.

The `strict` flag always coexists with unconditional rejection of unknown fields and unknown field IDs. It adds schema‑level validation only for clearly typed objects, allowing the same RPCs to be used both for raw codec testing (for example partial structures) and for format validation of full transactions or ledger entries.

Limiting the binary representation to hex only aligns with existing XRPL APIs, avoids extra encoding parameters, and keeps examples and tooling simple.

## 5. Backwards Compatibility

This XLS introduces new RPC methods only. Existing RPCs are unchanged, and no consensus‑critical behavior is modified. Nodes that do not implement these RPCs will return an unknown‑method error. Clients SHOULD NOT assume these methods are universally available and SHOULD handle their absence gracefully.

## 6. Security Considerations

`encode` and `decode` do not modify ledger state but can consume significant CPU and memory on large or adversarial inputs. Implementations SHOULD enforce reasonable limits on input size and complexity and SHOULD apply authentication and rate limiting, preferably restricting these RPCs to admin or otherwise privileged contexts.

`decode` may reveal how the server interprets experimental or in‑progress STypes. Operators deploying experimental features on private networks SHOULD consider whether exposing these RPCs externally matches their threat model. The `strict` flag provides format validation but is not a complete security check; downstream components MUST NOT treat it as a sufficient condition for economic or application‑level safety.
